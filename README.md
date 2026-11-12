```
#!/bin/sh

# Export a directory as one deterministic, restorable Markdown file.
# Python is used so filenames (including whitespace and newlines) and file bytes
# can be handled without relying on non-portable find/sort extensions.

set -u

usage() {
    printf 'Usage: %s TARGET_DIRECTORY\n' "${0##*/}" >&2
}

if [ "$#" -ne 1 ]; then
    usage
    exit 2
fi

if [ ! -d "$1" ]; then
    printf 'Error: target is not an existing directory: %s\n' "$1" >&2
    exit 2
fi

if ! command -v python3 >/dev/null 2>&1; then
    printf 'Error: python3 is required.\n' >&2
    exit 2
fi

exec python3 - "$1" <<'PY'
import base64
import json
import os
import stat
import sys
import tempfile


target_arg = sys.argv[1]
target = os.path.abspath(target_arg)
root_name = os.path.basename(target.rstrip(os.sep))
output_name = "export.md"
output_path = os.path.join(target, output_name)
errors = []

if not root_name:
    print("Error: exporting the filesystem root is not supported.", file=sys.stderr)
    sys.exit(2)


def path_bytes(path):
    return os.fsencode(path)


def sort_key(path):
    return path_bytes(path)


def rel(path):
    value = os.path.relpath(path, target)
    return "." if value == "." else value.replace(os.sep, "/")


def metadata(path, st, kind):
    result = {
        "path": rel(path),
        "type": kind,
        "mode": format(stat.S_IMODE(st.st_mode), "04o"),
        "uid": st.st_uid,
        "gid": st.st_gid,
        "mtime_ns": st.st_mtime_ns,
    }
    if kind == "file":
        result["size"] = st.st_size
    return result


entries = []


def scan(directory):
    try:
        with os.scandir(directory) as iterator:
            children = sorted(iterator, key=lambda item: sort_key(item.name))
    except OSError as exc:
        errors.append(f"cannot read directory {rel(directory)!r}: {exc}")
        return

    for item in children:
        path = item.path
        if directory == target and item.name == output_name:
            continue
        try:
            st = item.stat(follow_symlinks=False)
        except OSError as exc:
            errors.append(f"cannot stat {rel(path)!r}: {exc}")
            continue

        mode = st.st_mode
        if stat.S_ISLNK(mode):
            try:
                link_target = os.readlink(path)
            except OSError as exc:
                errors.append(f"cannot read symbolic link {rel(path)!r}: {exc}")
                continue
            data = metadata(path, st, "symlink")
            data["target"] = link_target
            entries.append((path, data))
        elif stat.S_ISDIR(mode):
            entries.append((path, metadata(path, st, "directory")))
            scan(path)
        elif stat.S_ISREG(mode):
            entries.append((path, metadata(path, st, "file")))
        else:
            errors.append(
                f"unsupported special file {rel(path)!r} "
                f"(mode {format(mode, '#o')})"
            )


root_st = os.stat(target, follow_symlinks=False)
root_metadata = metadata(target, root_st, "directory")
root_metadata["root_name"] = root_name
entries.append((target, root_metadata))
scan(target)
entries.sort(key=lambda pair: sort_key(rel(pair[0])))

# Read all source files before replacing export.md. This both detects failures
# early and ensures the final file is not installed when the export is partial.
contents = {}
for path, data in entries:
    if data["type"] != "file":
        continue
    try:
        with open(path, "rb") as source:
            contents[path] = source.read()
    except OSError as exc:
        errors.append(f"cannot read file {rel(path)!r}: {exc}")

if errors:
    for message in errors:
        print(f"Error: {message}", file=sys.stderr)
    print("Error: export is incomplete; export.md was not replaced.", file=sys.stderr)
    sys.exit(1)

header = """# Directory export

> **Sensitive data warning:** This export contains the source files verbatim and
> may include credentials, private keys, tokens, or other confidential data.

## Format

This file was produced by `get_config.sh`. Entries are ordered by relative path
using bytewise filename order. Hidden files and empty directories are included.
The generated `export.md` is excluded. The script creates a temporary
`.get_config-export-*.tmp` file only after the scan is complete, then atomically
renames it to `export.md`; that temporary output is also excluded. No other paths
are intentionally excluded.

The folder structure is a JSON Lines manifest. Each line records a relative
path, type, POSIX mode, numeric owner and group, and nanosecond modification
time. The root entry records its original folder name in `root_name`.
Symbolic-link targets are stored in `target` and are never followed.

File records repeat the manifest metadata. UTF-8 text without NUL bytes uses
`encoding: raw-utf8` and is stored as exactly `stored_bytes` bytes after the
`<!-- Data -->` line; this length-delimited representation safely permits any
Markdown, including code fences and marker-like text. Other files use
`encoding: base64`; decoding yields exactly `original_bytes` bytes. A single LF
separator follows each payload and is not part of it.

## Restoration

Parse each JSON manifest line to create directories first. Then parse each file
record's JSON metadata, read exactly `stored_bytes` bytes after its data marker,
and either write those bytes directly (`raw-utf8`) or Base64-decode them
(`base64`). Create symbolic links with their recorded targets, without resolving
them. Finally apply modes with `chmod`, modification times with `utime`, and,
where permitted, numeric owners/groups with `chown`; apply directory metadata
last. Verify each restored file's byte count. Root privileges may be needed to
restore ownership exactly.

## Dependencies and usage

Requires a POSIX-compatible shell and Python 3. Run from any directory:

    ./get_config.sh /path/to/repository
    ./get_config.sh "../path with spaces"

## Folder structure (JSON Lines)

"""

fd, temporary_path = tempfile.mkstemp(
    prefix=".get_config-export-", suffix=".tmp", dir=target
)
try:
    with os.fdopen(fd, "wb") as out:
        out.write(header.encode("utf-8"))
        for _path, data in entries:
            out.write(json.dumps(data, ensure_ascii=True, sort_keys=True).encode("utf-8"))
            out.write(b"\n")

        out.write(b"\n## File contents\n\n")
        for path, data in entries:
            if data["type"] != "file":
                continue
            raw = contents[path]
            try:
                raw.decode("utf-8")
                is_text = b"\0" not in raw
            except UnicodeDecodeError:
                is_text = False
            payload = raw if is_text else base64.b64encode(raw)
            record = dict(data)
            record.update({
                "encoding": "raw-utf8" if is_text else "base64",
                "original_bytes": len(raw),
                "stored_bytes": len(payload),
            })
            comment_path = (
                rel(path)
                .replace("\\", "\\\\")
                .replace("\r", "\\r")
                .replace("\n", "\\n")
                .replace("--", "\\u002d\\u002d")
            )
            out.write(f"<!-- File: {comment_path} -->\n".encode("utf-8", "backslashreplace"))
            out.write(b"<!-- Metadata: ")
            out.write(json.dumps(record, ensure_ascii=True, sort_keys=True).encode("utf-8"))
            out.write(b" -->\n<!-- Data -->\n")
            out.write(payload)
            out.write(b"\n<!-- End file -->\n\n")
        out.flush()
        os.fsync(out.fileno())
    os.replace(temporary_path, output_path)
except BaseException:
    try:
        os.unlink(temporary_path)
    except OSError:
        pass
    raise

print(f"Created {output_path}")
PY

```
