# Carracho Server: Changelog

Versionsnummern und Veröffentlichungen des Servers sind vom Client unabhängig.

## 1.1.7 (Build 18)

### Critical: prevent account loss when changing the administrator password

Fixed a data-loss defect in the macOS Server administration GUI. When the Server GUI and the background `carracho-serverd` service had independently loaded the same `server.db`, the GUI could retain an outdated account snapshot. Changing the administrator password then saved that stale *entire* server state, replacing the current accounts table and removing accounts created or changed by the daemon since the GUI opened. The updated code applies the password change against the latest database state, preserving all other accounts, credentials, newsgroups and server settings.

### Atomic cross-process SQLite state updates

Every Server state mutation now acquires SQLite's `BEGIN IMMEDIATE` writer lock **before reading the current persisted state**, applies the change to that latest state and commits it atomically. This prevents the independently running Server GUI and system-service daemon from overwriting one another's newer data. An error rolls back the whole transaction. The old API for blindly saving an out-of-date full-state snapshot has been removed.

### Administrator password changes update credentials only

The Server app now uses the backend's dedicated password-change operation rather than submitting its cached administrator account as an entire replacement record. Existing admin metadata, group assignments, profile settings and changes made through a second process remain intact. The normal new-password validation and authentication verifier storage are unchanged.

### Refuse silent database reinitialization on existing installations

An existing `server.db` without a valid persisted Server state is now treated as an error rather than silently replaced with fresh default accounts. First-run initialization is serialized inside the same SQLite write lock so two simultaneous process starts cannot independently bootstrap the database. Fresh installations still initialize normally. This protection **does not recover** accounts already lost to an earlier overwritten state; restore those from a database backup if available.

### Regression coverage for GUI and daemon account safety

Added a standalone SQLite regression test using two separate backend instances pointing to one disposable test database. It confirms that an administrator password change from a stale GUI snapshot preserves accounts created by the other process, including their logins, UUIDs and usable passwords. Tests also check preserved newsgroups and server settings, reverse-direction stale-state changes, rollback after a deliberate failure and safe handling of an incomplete existing database.

### Update the macOS system-service daemon as well as the GUI

The fix changes shared persistence code used by **both** the macOS Server app and its embedded `carracho-serverd` helper. After installing the new app, use **System Service → Update Now** to replace any previously installed system-service helper. Installing only the GUI while leaving an older daemon running does not fully protect the Server database. No SQLite schema migration or Classic/modern packet-format change is required. The shared Server version metadata is also raised to 1.1.7 (build 18) for native Linux build/package reporting, but the reported stale-snapshot bug concerns the macOS Server GUI and daemon.

## 1.1.6 (Build 17)

### Dedicated Server and Tracker system services

The macOS Server and Tracker can now run as two independent launchd system services. Both jobs use the same compact carracho-serverd helper embedded in the Server app, while Server and Tracker retain separate launchd jobs and separate running/automatic-start state.

Installing a service copies only the headless daemon binary and the service definitions required to run it. The daemon directory no longer needs a second complete copy of **Carracho Server.app**, avoiding duplicated application bundles under /Applications and the service installation.

The helper is built as a Universal 2 binary and contains the headless Server/Tracker runtime rather than the AppKit administration UI or Sparkle updater.

### System Service is the single control surface

Server and Tracker runtime control is now consolidated under the always-visible **System Service** section for each component. The old duplicate Start/Stop controls in the page header, application menu and status menu have been removed.

Each service can be installed or uninstalled independently, started or stopped independently, and configured independently for automatic startup with macOS. Current running state and startup-at-boot state are separate: disabling automatic startup does not stop a service that is already running.

The **System Service** section is permanently expanded and no longer has a disclosure arrow, so installation and runtime controls remain visible without another layer of navigation.

### Daemon update reminder after an app update

The Server app now compares the daemon helper embedded in the current app with the binary installed for the system services. The comparison is byte-for-byte, so rebuilt hotfixes are detected even if their marketing version happens to be unchanged.

When an installed daemon differs from the helper in the updated app, **Reinstall** changes to **Update Now**. On every normal launch of the Server app, a warning explains that the application was updated and the system service must also be refreshed. The warning continues to appear until the installed daemon matches the current app.

Choosing **Update Now** performs the same privileged service refresh used by the System Service controls while preserving the installed sibling service's running and automatic-start state. Because Server and Tracker share the daemon binary, updating either installed service refreshes the common executable.

## 1.1.5 (Build 16)

### Paged directory replies for modern clients

The Swift/macOS Server and native Linux Server now support modern directory-list paging. Each page stays below the legacy UInt16 TLV-value limit and includes a continuation offset when more entries remain.

Paging is requested explicitly by modern clients. Clients that do not request it continue to receive the historical single-listing response, leaving Classic packet layouts unchanged.

### Shared support for boot-scoped client history

No new durable identity is invented for old peers that cannot provide one. Instead, the client uses the existing server-uptime information to scope numeric user IDs to one server process lifetime. Current servers continue to provide stable account UUID metadata to modern clients as introduced in 1.1.4.

This keeps the 1.1.5 fallback compatible with original/older servers while preserving the 1.1.4 protection against history being rebound after a server restart.

## 1.1.4 (Build 15)

### Stable account identity for modern peer metadata

Both the Swift/macOS server and the native Linux server now include the authenticated account UUID in modern user-arrival and user-update metadata. The stable identifier is sent in initial user snapshots, new-user events and later user metadata refreshes so modern clients can distinguish account identity from the temporary numeric session user ID. Modern observers also receive the stable account identity for Classic peers, while Classic recipients continue to receive the historical packet layouts unchanged.

This identity metadata does not change Private Message routing or grant access to another user's messages. Messages are still routed to the current live session; the UUID is used by modern clients to associate local history with the correct authenticated account.

## 1.1.3 (Build 14)

### Secure first-run administrator credentials

A genuinely new server database no longer leaves the built-in `admin` account with an empty password until an administrator changes it manually. Before the server runtime can accept connections, Carracho generates a random 128-bit initial password and applies it to the built-in `admin` account.

On macOS, the Server app displays the generated password once in an **Initial Server Credentials** warning and provides a **Copy Password** action. On Linux source installs and fresh Debian package installs, the installer prints the initial administrator login and password before the service is started. In both cases the administrator is explicitly told to change the password immediately after the first login.

The generated plaintext password is not written to a separate credentials file. Existing databases and migrated legacy state are not assigned a new password by this bootstrap logic.

### Anonymous first-run notice

The built-in `anonymous` Guest account intentionally remains passwordless by default. The first-run notice now states this explicitly and explains that an administrator can set a password for `anonymous` under **Administration → Accounts** when passwordless Guest access is not desired.

### Linux runtime permissions

The Linux installation instructions now explicitly state that the user configured to run `carracho-server` must have full read, write, create/remove, and directory-traversal access to the complete `/opt/carracho` tree.

With the standard service this user is `carracho`, so `/opt/carracho` should remain owned appropriately by `carracho:carracho`. Installations using a different systemd `User=` or `Group=` must adjust ownership and permissions accordingly. The documentation explicitly discourages making `/opt/carracho` world-writable.
