# Samba File Server with Granular Access Control

A practical guide to running a company file server on **Ubuntu + Samba**. Every employee gets a private share, managers control everything below them in the org chart, and IT admins can reach everything. It is built on POSIX ACLs and Samba share rules, with no extra software.

This is the setup used in production at a creative agency with several departments. All names, usernames, paths and IP addresses below are examples.

## What this setup gives you

- Every employee has a dedicated personal folder
- Employees can upload and manage files only in their own folder
- Managers have full control (including delete) over their entire subtree
- Superadmins have unrestricted access to the entire tree
- No unauthorized user can see or access another department's data

## 1. Base configuration

| Item | Value |
|---|---|
| Root path | `/data-disk/Company` |
| Operating system | Ubuntu Linux |
| File sharing service | Samba (`smbd` / `nmbd`) |
| Access control | POSIX ACLs + Samba `valid users` + `force group` |
| Server IP (example) | `192.168.1.10` |

Install the basics:

```bash
sudo apt install samba acl smbclient
```

## 2. Directory structure

The folder tree mirrors the company org chart:

```text
/data-disk/Company/
├── Departments/
│   ├── Marketing/
│   │   └── Project_Managers/
│   │       └── Alice_Brown/
│   │           └── Assistant_Project_Manager/
│   │               └── Bob_Clark/
│   │                   └── 3D_Designer/
│   │                       └── Carol_Davis/
│   │                           └── Video_Editors/
│   │                               └── Dan_Evans/          ← Employee folder
│   │
│   └── Production/
│       └── Project_Managers/
│           └── Frank_Green/
│               └── Assistant_Managers/
│                   └── Grace_Hall/
│                       ├── Video_Editors/
│                       │   ├── Henry_King/
│                       │   ├── Ivy_Lee/
│                       │   └── ...
│                       ├── Graphic_Designer/
│                       │   ├── Jack_Moore/
│                       │   ├── Kate_Nash/
│                       │   └── ...
│                       └── Copywriters/
│                           └── Liam_Owen/
│
└── (Other departments follow the same pattern)
```

## 3. Users and groups

| Role | Examples | Linux group |
|---|---|---|
| Employee | `dan_evans`, `henry_king`, `ivy_lee`, `jack_moore`, `kate_nash`, `liam_owen` | `marketing` / `production` |
| Assistant Manager | `grace_hall`, `carol_davis`, `bob_clark` | `marketing` / `production` |
| Project Manager | `frank_green`, `alice_brown` | `marketing` / `production` |
| Superadmins | `it_admin1`, `it_admin2`, `it_admin3` | `superadmins` |

Every Samba user gets a `nologin` shell, so they cannot SSH into the server:

```bash
sudo useradd -M -d <full_path_to_folder> -s /usr/sbin/nologin <username>
sudo usermod -aG <department_group> <username>
```

## 4. Permission model

Every employee folder is set up with these steps, in order:

1. Create the Linux user (`nologin` shell, home = share path)
2. Create the directory with `mkdir -p`
3. Give the employee access along the path so they can reach their folder (see [Hardening](#8-hardening-tips))
4. Grant managers and `@superadmins` `rwx` on the employee folder
5. Set ownership: `chown -R user:department_group`
6. Set base permissions: `chmod -R 770`
7. Remove restrictive attributes (`chattr -R -a` and `-i`) so managers can delete

### Access matrix

| Person | Can read / write | Can delete |
|---|---|---|
| Employee (own folder) | Yes | Yes |
| Direct manager | Yes | Yes |
| Higher managers | Yes | Yes |
| Superadmins | Yes (everywhere) | Yes |
| Other employees | No | No |

## 5. Samba share configuration

Each employee gets a dedicated share in `/etc/samba/smb.conf`:

```ini
[Dan_Evans]
   path = /data-disk/Company/Departments/Marketing/.../Video_Editors/Dan_Evans
   valid users = dan_evans carol_davis bob_clark alice_brown @superadmins
   read only = no
   browseable = yes
   writable = yes
   create mask = 0777
   directory mask = 0777
   force group = marketing
```

After adding or changing a share:

```bash
sudo smbpasswd -a <username>
sudo testparm && sudo smbcontrol all reload-config
```

## 6. Standard command sequence

This is the full sequence for adding one employee:

```bash
# 1. Create user
sudo useradd -M -d <full_path> -s /usr/sbin/nologin <username>
sudo usermod -aG <dept_group> <username>

# 2. Create directory
sudo mkdir -p <full_path>

# 3. ACLs – path access for the employee + managers + superadmins
sudo setfacl -m u:<username>:x <each_parent_folder>          # traverse only, not recursive
sudo setfacl -R -m u:<username>:rwx <employee_folder>
sudo setfacl -R -m u:<manager>:rwx <employee_folder>        # repeat for each manager up the chain
sudo setfacl -R -m g:superadmins:rwx <employee_folder>

# 4. Ownership & permissions
sudo chown -R <username>:<dept_group> <employee_folder>
sudo chmod -R 770 <employee_folder>

# 5. Remove restrictive attributes (important for delete rights)
sudo chattr -R -a <employee_folder>
sudo chattr -R -i <employee_folder>

# 6. Samba
# → add the share block to /etc/samba/smb.conf
sudo smbpasswd -a <username>
sudo smbcontrol all reload-config
```

## 7. Testing access

From the server or any client:

```bash
smbclient //192.168.1.10/<ShareName> -U <username>
# or
smbclient //localhost/<ShareName> -U <username>
```

Once connected you can use `ls`, `put`, `get`, `mkdir`, `rmdir`, `rm` and so on. On Windows, open `\\192.168.1.10\<ShareName>` in File Explorer.

## 8. Hardening tips

- **Keep parent-folder access to traverse only.** Running `setfacl -R -m u:<user>:rwx` on a *parent* folder gives that user read/write on everything below it, including colleagues' folders, at the filesystem level. Samba's `valid users` still keeps them out of other shares, but the files themselves are no longer protected. An employee only needs `x` (traverse) on each parent folder, applied without `-R`.
- **Add default ACLs** so new files and folders inherit the right permissions: `setfacl -R -d -m u:<manager>:rwx <employee_folder>`.
- **Tighten the masks.** `create mask = 0770` and `directory mask = 0770` stop new files from getting "other" permissions.
- **Hide other people's shares** with `access based share enum = yes` in `[global]`, so users only see the shares they can open.

## 9. Best practices

- Always use the full absolute path when creating users and setting ACLs.
- Always run `chattr -R -a` and `chattr -R -i` so managers and superadmins can delete files.
- After any change to `smb.conf`, run `sudo testparm && sudo smbcontrol all reload-config`.
- Employee accounts use `/usr/sbin/nologin`, so they cannot log in via SSH.
- The `force group` directive makes all new files inherit the department group.
- Superadmins are referenced as `@superadmins` in every `valid users` line.

## Contributing

Spotted a mistake or have a better approach? Open an issue or a pull request. If this guide helped you, please give it a ⭐.

## License

[MIT](LICENSE)
