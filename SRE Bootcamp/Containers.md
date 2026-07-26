In containers, **UID** (User ID) and **GID** (Group ID) are ==the numeric identifiers the Linux kernel uses to determine process privileges and file ownership==. The **UID** identifies the specific user running the application, while the **GID** identifies their primary group.

Key Concepts

- **Identifiers, Not Names:** Internally, the Linux kernel relies exclusively on numbers, mapping them to text strings (like `root` or `nginx`) via `/etc/passwd`.
- **Default State:** By default, many containers execute processes as the `root` user, corresponding to **UID 0** and **GID 0**.
- **Host Mapping:** Without user namespaces enabled, **UID 0** inside the container maps to the **UID 0 (root)** on the host machine. A non-root container UID maps to the exact same number on the host, which can lead to file-permission conflicts if the host already uses that ID.


