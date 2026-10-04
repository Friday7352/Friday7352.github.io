# Clearspace file locking

Implemented October 1, 2026 for CS 499 Milestone Four. This completes the file-lock
storage portion of the original database plan and adds the corresponding lock and
unlock feature originally proposed under software engineering.

## Using the feature

Right-click one file and select **Lock file with password**. The first lock sets a
master password, confirmed twice, with 12 to 1,024 characters. Later locks require
that same password. Right-click **Unlock file**, or use Open on a locked file, to
restore its readable contents. Cancellation before replacement keeps the original.
The password cannot be recovered or reset through this feature.

The initial version accepts one writable file up to 64 MiB (67,108,864 bytes) on a
local NTFS drive. It rejects folders, read-only files, hard links, reparse points,
linked ancestor folders, offline/cloud placeholders, Windows-encrypted files, and
files with alternate data streams. This includes downloaded files with a
Zone.Identifier stream. It does not remove streams to make a file eligible.
The filename and extension remain unchanged. Password changes and bulk locking
are outside this version's scope.

## Encryption and password handling

`FileLockFormat` uses .NET AES-256-GCM with a randomly generated 32-byte data key
for each lock operation. A password-derived 32-byte key wraps that data key using
a separate AES-GCM operation. Both authentication tags are 16 bytes. The wrapping
and content operations use independent keys and fresh 12-byte nonces. Password
derivation uses PBKDF2-HMAC-SHA256, a random 32-byte salt, and 600,000 iterations.
The parser accepts only that work factor and the supported size limit, avoiding
unbounded work requested by an untrusted header.

The 156-byte versioned header contains the magic `CSLOCK01`, a random file ID,
work factor, plaintext length, salt, wrapping nonce, wrapped data key, wrapping
tag, content nonce, and content tag. The wrapping operation authenticates the
header prefix through the salt. The content operation authenticates the header
through the content nonce. Changing header fields, ciphertext, length, or tags
causes rejection. Paths are deliberately excluded so moving or renaming an
encrypted file does not prevent recovery.

Password entry uses WPF PasswordBox. The UI copies its SecureString into a
short-lived character array, clears both password controls, and clears the array
after completion. The service clears application-owned plaintext and unwrapped
key arrays in finally blocks. This is not a guarantee that operating-system
paging or every runtime/library copy is erased.

## SQLite schema and migration

The existing `tags.db` now uses schema version 2. The upgrade from version 1 adds
two tables inside a transaction and preserves Tags, Paths, and PathTags. A fresh
database imports legacy JSON and installs both schema versions in one transaction.
An invalid import still rolls back the entire schema. Future versions are rejected.

- `LockSettings` has a single row containing a random salt, the work factor, and
  an HMAC verifier derived from the master password. It stores no plaintext password.
- `LockedFiles` maps case-insensitive FilePath values to FileId, the binary
  authenticated Header, State, and UpdatedUtc. The header contains only a wrapped
  data key. An index on FileId supports identifying records for the same envelope.
- All values are bound as SQL parameters. Constraints enforce valid header and
  verifier lengths and the three supported operation states.

The header is deliberately duplicated in the file. The database manages local
metadata, while an encrypted file remains portable and recoverable using its
password on a fresh installation. If the file is renamed outside the app, an old
path record can remain; the file header determines whether the file is locked.
Unlocking one copy removes only its path record, not records for other copies.

## Failure handling and boundaries

1. Validate the path and file, reject unsupported links/streams, and open a handle
   that denies concurrent writers.
2. Encrypt, or authenticate and decrypt, in memory. Authentication failure causes
   no file replacement. The size limit bounds the memory cost of this approach.
3. Commit a Preparing or Unlocking database record before replacing any file.
4. Write a uniquely named temporary file beside the source. Its ACL allows only
   the current Windows user and SYSTEM. Flush it to disk and verify its hash.
5. Recheck the target identity and eligibility, then use File.Replace. Preserve
   the source timestamp and require successful metadata handling.
6. Record Locked, or remove the path record after unlock. If this last database
   update fails, report that the file operation succeeded with a metadata warning.

The filesystem replacement and SQLite commit are separate operations. The design
does not claim a transaction across both systems. The embedded authenticated
header makes a completed replacement recoverable when the final database update
does not occur. A retry reads the actual file contents rather than trusting a
possibly stale state value.

Ordinary errors and cancellation delete the operation's temporary file. A process
or machine crash during unlock preparation can leave a plaintext temporary file
with the restricted ACL above; automatic crash-remnant cleanup is not implemented.
The same Windows account or an administrator can access that remnant. Replacement
also does not securely erase old disk blocks, existing backups, cached previews,
search snippets, or synchronized copies. File names remain visible. Use trusted
local directories: identity checks do not establish a security boundary against
an adversary who can continuously replace directory entries. This feature protects
the current file contents at rest; it is not a substitute for full-disk encryption.

## Verification

The full Release suite passed 251 tests on October 1, 2026. `FileLockTests` includes
empty and nonempty round trips, independent keys/nonces, wrong passwords, modified
headers and ciphertext, truncation, renamed files with a fresh database, schema
upgrade, cancellation, busy files, hard links, additional streams, size limits,
and failures around replacement and database writes. Interruption tests inject
exceptions at durable boundaries; they do not simulate a physical power loss.

A separate WPF dialog check used disposable files and a private test database.
The actual lock and unlock buttons restored the original bytes; an incorrect
password showed an error while retaining the encrypted file. Both dialog layouts
were rendered and inspected. Release publishing succeeded. No personal files
were encrypted during these checks. The interface still merits a user walkthrough
with disposable files before use on important data.

## References

- Microsoft, [AesGcm.Encrypt](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.aesgcm.encrypt?view=net-10.0).
- Microsoft, [FileSystemAclExtensions.Create](https://learn.microsoft.com/en-us/dotnet/api/system.io.filesystemaclextensions.create?view=net-10.0).
- Microsoft, [ReplaceFile](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-replacefilea).
- OWASP, [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html).
- OWASP, [Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html).
