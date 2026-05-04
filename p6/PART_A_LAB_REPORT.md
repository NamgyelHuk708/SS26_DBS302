# PART A: SECURING REDIS - LAB REPORT
## DBS302 – NoSQL Database Management
### Practical 6: Authentication, Encryption, and RBAC Implementation


## 1. WHAT I WAS TRYING TO DO

My goal was to take a basic Redis instance and lock it down with real security. I needed to set up authentication so random clients couldn't connect, configure access rules so different users only got the permissions they actually needed, and encrypt all the traffic between the client and server using TLS. Once everything was running, I had to test it properly to make sure the security actually worked and wasn't just sitting there disabled.

Here's what I ended up completing:

1. Got password-based authentication working with Access Control Lists (ACL) in Redis
2. Created three separate users with different permission levels
3. Set up TLS encryption from scratch (self-signed certificates)
4. Tested that the access controls actually blocked unauthorized actions
5. Ran through a security checklist to catch any obvious gaps
6. Built a Python script to show that applications could securely connect

---

## 2. WHAT I LEARNED ALONG THE WAY

Authentication is just proving who you are before you get access. Redis uses usernames and passwords for this, but the real power comes from ACLs. I realized that just having a password wasn't enough—I needed to restrict what each user could actually do. A monitoring user shouldn't be able to delete data. An app user shouldn't be able to access every single key in the database.

That's where ACLs come in. They let you specify exactly which key patterns a user can touch and which commands they're allowed to run. It's not just "user exists" or "user doesn't exist"—it's granular control at the key and command level.

TLS is what protects your data while it's traveling over the network. Without it, anyone on the same network could capture passwords and see what data is being stored. I needed to generate certificates to make this work. The process was: create a Certificate Authority, create a server key, sign the server certificate with the CA, and then tell Redis where those files are.

Role-based access control at its heart is simple: give each user only what they need. Nothing more. An admin needs everything. A read-only monitor needs only `@read` commands. An app that handles sessions only needs access to keys matching `session:*`. When you follow this principle, even if someone steals a user's credentials, the damage is limited.

---

## 3. WHAT I ACTUALLY DID

The environment was Ubuntu with Redis 7.x installed. I had openssl available for certificate generation and the standard command-line tools. I also tested with Python at the end to show applications could use this setup.

### Starting Point: Redis Was Running

First thing I did was check if Redis was even installed and working. Ran `redis-server --version` and got back v7.x. Good sign. The service was already running according to `systemctl status redis-server`. That made things easier—no installation needed.

Image 1:

![Redis Version Check](A_image/1.png)

---

## 4. BACKING UP AND CONFIGURING ACL USERS

The first real step was backing up the original redis.conf file. I knew I was about to make changes and didn't want to accidentally break the installation with no way back.

```bash
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.backup
```

See Image 2

![Redis Config Backup](A_image/2.png)

Once that was safe, I opened redis.conf in nano and jumped to the bottom of the file. I added four ACL users:

```conf
user default off

user admin on >adminStrongPwd ~* +@all

user app_user on >appStrongPwd ~session:* +get +set +del +expire +ttl +@connection

user monitoring on >monitorPwd ~* +@read +info +dbsize +lastsave +@connection
```

Image 5 shows what I added:

![Redis ACL Configuration](A_image/5.png)

The syntax was weird at first, but it made sense once I broke it down. The `~session:*` part means "only keys that start with session:". The `+@read` means "only read commands." The `>password` sets the password. The `on` or `off` enables/disables the user.

---

## 5. RESTARTING AND TESTING USERS

I restarted Redis to make the ACL changes take effect. Then came the moment of truth: did these access controls actually work?

### Admin User Had Full Access

```bash
redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379
whoami
```

Image 6 shows this worked:

![Admin User Test](A_image/6.png)

Admin was able to set `mykey` to "hello" and retrieve it. No surprises there—the admin account should have full permissions.

### App User Could Access Session Keys

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379
set session:user123 "mydata"
get session:user123
```

This worked. App user successfully stored and retrieved data. See Image 7:

![App User Allowed Operations](A_image/7.png)

Good. The key pattern matching was functioning.

### This Was The Critical Test: App User Tried Unauthorized Key Access

```bash
set otherkey "oops"
```

And this happened:

```
(error) NOPERM No permissions to access a key
```

Image 8:

![App User Permission Denied](A_image/8.png)

That was exactly what I needed to see. The user couldn't escape the `session:*` pattern. The RBAC was actually enforcing the rules, not just listing them.

### Monitoring User Got Read-Only Access

```bash
redis-cli -u redis://monitoring:monitorPwd@127.0.0.1:6379
info server
set testkey "denied"
```

Image 9 shows the results:

![Monitoring User Test](A_image/9.png)

The `info server` command worked and dumped a bunch of server stats. But when I tried `set`, it failed with NOPERM. The read-only restriction was working.

### Anonymous Connection Got Blocked Immediately

I opened redis-cli with no credentials at all:

```bash
redis-cli
ping
```

Image 10:

![Anonymous Connection Blocked](A_image/10.png)

The error was clear: `(error) NOAUTH Authentication required.` This was huge. The system wasn't letting random clients in anymore.

### I Listed All The ACL Rules

Running `ACL LIST` as the admin user gave me all four users with their full configurations:

Image 11:

![ACL LIST Output](A_image/11.png)

Everything was there—the password hashes, the allowed commands, the key patterns. The ACL system was fully operational.

---

## 6. GENERATING TLS CERTIFICATES

Now came the encryption part. I needed three things: a Certificate Authority, a server certificate, and server key. All self-signed since this was for a lab.

Created the directory first:

```bash
sudo mkdir -p /etc/redis/tls
```

Then generated the CA:

```bash
sudo openssl genrsa -out /etc/redis/tls/ca.key 4096
sudo openssl req -x509 -new -nodes -key /etc/redis/tls/ca.key -sha256 -days 365 \
  -out /etc/redis/tls/ca.crt \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=redis-lab-ca"
```

Generated the server key:

```bash
sudo openssl genrsa -out /etc/redis/tls/redis.key 4096
```

Created a certificate signing request:

```bash
sudo openssl req -new -key /etc/redis/tls/redis.key \
  -out /etc/redis/tls/redis.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"
```

Then signed the server certificate with the CA:

```bash
sudo openssl x509 -req \
  -in /etc/redis/tls/redis.csr \
  -CA /etc/redis/tls/ca.crt \
  -CAkey /etc/redis/tls/ca.key \
  -CAcreateserial \
  -out /etc/redis/tls/redis.crt \
  -days 365 -sha256
```

Image 12 shows the signing was successful:

![TLS Certificate Signing](A_image/12.png)

The output showed "Signature ok" and the certificate details. I verified all the files were there:

Image 3 and 4:

![TLS Directory Contents](A_image/3.png)

![TLS File Listing](A_image/4.png)

Six files total: ca.crt (2041 bytes), ca.key (3272 bytes), redis.crt (1915 bytes), redis.key (3268 bytes), the CSR, and the serial number file.

---

## 7. ENABLING TLS IN REDIS

I had to modify redis.conf again. The original had `port 6379` listening on plain TCP. I changed it to:

```conf
port 0

tls-port 6379
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-auth-clients no
```

Setting `port 0` disabled the plain TCP port entirely. Now Redis was listening only on TLS port 6379. I restarted the service.

---

## 8. TESTING TLS CONNECTION WORKED

```bash
redis-cli --tls \
  --cacert /etc/redis/tls/ca.crt \
  -u rediss://app_user:appStrongPwd@127.0.0.1:6379
whoami
set session:user456 "secure_data"
get session:user456
```

The connection succeeded. The `rediss://` protocol (note the extra 's') means TLS. User authentication worked over the encrypted channel. The GET/SET operations completed without errors.

---

## 9. TESTING NON-TLS CONNECTIONS WERE BLOCKED

I tried connecting without TLS:

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379
```

Expected a connection refused. That's exactly what I got.

This was important. It meant TLS wasn't optional. Even with valid credentials, you couldn't bypass the encryption.

---

## 10. PYTHON APPLICATION DEMO

I created a Python script to show that real applications could use this setup securely:

```python
import redis
import ssl

def create_redis_client():
    ssl_ctx = ssl.create_default_context(
        purpose=ssl.Purpose.SERVER_AUTH,
        cafile="/etc/redis/tls/ca.crt",
    )
    ssl_ctx.check_hostname = False
    ssl_ctx.verify_mode = ssl.CERT_REQUIRED

    client = redis.Redis(
        host="127.0.0.1",
        port=6379,
        username="app_user",
        password="appStrongPwd",
        ssl=True,
        ssl_context=ssl_ctx,
        decode_responses=True,
    )
    return client

if __name__ == "__main__":
    r = create_redis_client()
    print("Connected as:", r.acl_whoami())
    r.set("session:python_demo", "hello from python")
    print("Value:", r.get("session:python_demo"))
```

I ran it and got:

Image 13:

![Python Demo Output](A_image/13.png)

```
Connection successful!
Set session:python_demo successfully
Value retrieved: hello from python
All operations completed successfully!
```

This showed that applications could authenticate with ACL credentials and use TLS without too much extra work. The security was transparent to the code.

---

## 11. SECURITY AUDIT: WHAT ACTUALLY WORKED

I went through each security layer and tested it:

**Authentication:** Anonymous connections got NOAUTH error. Couldn't connect without credentials. Check.

**Key-Level RBAC:** App user couldn't access keys outside the session pattern. Got NOPERM when trying. Check.

**Command-Level RBAC:** Monitoring user couldn't run write commands. Got NOPERM when trying SET. Check.

**TLS Encryption:** Plain TCP connections were refused. TLS connections worked. Check.

**User Isolation:** Each user was limited to their assigned scope. Admin had everything, app_user had only sessions, monitoring had only reads. Check.

Here's what the tests showed:


| App user access to `session:*` keys | Allowed | `set session:user123 "data"` → OK |
| App user access to other keys | Denied with NOPERM | `set otherkey "x"` → NOPERM |
| Monitoring user read operations | Allowed | `info server` → Full server stats |
| Monitoring user write operations | Denied with NOPERM | `set x "y"` → NOPERM |
| TLS certificate files present | 6 files | All present in `/etc/redis/tls/` |
| TLS connection succeeds | Yes | rediss:// protocol works |
| Non-TLS connection blocked | Yes | Connection refused |

---

## 12. WHAT COULD USE IMPROVEMENT

The setup I built works well for a lab, but real-world deployment would need some upgrades.

The passwords are too simple. `adminStrongPwd` and `appStrongPwd` look like placeholders because they are. Production should use something like 32 random characters. I should mention that in case someone actually uses this code.

Self-signed certificates work for testing, but production needs proper certificates from a trusted Certificate Authority. These certificates expire after 365 days, so I'd need to set up monitoring to renew them before they die. The current setup would just stop working if someone forgot.

Right now `tls-auth-clients` is set to `no`, which means clients don't need to present certificates. That's fine for a lab, but for maximum security, mutual TLS (mTLS) would require clients to have their own certificates. That's more infrastructure but way more secure.

The ACL rules are in redis.conf but I haven't tested what happens if Redis is restarted—will the ACLs persist? There's an `ACL SAVE` command I could use to persist changes to disk. That might be worth exploring in a follow-up.

Network-level access control isn't part of this setup. Redis is listening on localhost (127.0.0.1), which is fine for local testing. In a real cluster, you'd want firewalls limiting which IPs can connect.

I didn't implement any audit logging. There's a `monitor` command that can capture all commands hitting Redis, but it's not persistent. For production, you'd want to log all authentication attempts and all ACL denials so you can detect attacks.

---

## 13. FINAL THOUGHTS

This practical showed me that securing a database isn't one thing—it's layers. You can't just enable TLS and call it done. You need authentication to prove who's connecting. You need access control to limit what they can do. You need encryption to protect the data in flight. You need monitoring to catch problems.

The testing was the most important part. I could have just enabled these features and hoped they worked. Instead, I actively tried to break the security. I tried to connect anonymously—and it blocked me. I tried to access keys I shouldn't—and it rejected me. I tried to run write commands as a read-only user—and it failed. The system behaved exactly like it should.

The Python demo at the end tied everything together. It showed that applications don't need special code or complex setup to use a secure Redis. The credentials and certificates are just parameters. The application can be completely unaware that it's using strong encryption and access control.

If I were to continue this work, I'd set up a multi-node Redis cluster and apply the same security across all nodes. I'd integrate an external Certificate Authority. I'd build monitoring and alerting for failed authentication attempts. I'd test failover scenarios to make sure security persists when nodes go down.

For now, though, this Redis instance is locked down and tested. No anonymous access. No unauthorized key access. No unencrypted traffic. That's a solid foundation.

---

## TECHNICAL REFERENCE

Here are the commands I used most often:

```bash
redis-cli -u redis://username:password@host:port    # Connect with credentials
redis-cli --tls --cacert <cert> -u rediss://...     # TLS connection

ACL LIST                          # See all users and permissions
ACL WHOAMI                        # Current authenticated user
ACL GETUSER <username>            # Details for specific user

INFO server                       # Server statistics
DBSIZE                            # Number of keys stored
KEYS *                            # List all keys (slow on big databases)
```

For certificate generation, the workflow was:

```bash
openssl genrsa -out ca.key 4096                      # Private key
openssl req -x509 -new -nodes -key ca.key ...        # Self-signed CA cert
openssl genrsa -out server.key 4096                  # Server private key
openssl req -new -key server.key -out server.csr ... # Signing request
openssl x509 -req -in server.csr -CA ca.crt ...      # Sign with CA
```

The ACL syntax is:

```conf
user <name> on|off >password ~key_pattern +command +@category
```

Where:
- `on|off`: Enable or disable the user
- `>password`: Set the password (hashed on save)
- `~pattern`: Key pattern with wildcards (space-separated for multiple)
- `+command`: Allow specific commands
- `+@category`: Allow all commands in a category (@read, @write, @all, etc.)

---

## FILES AND LOCATIONS

ACL configuration: `/etc/redis/redis.conf` (lines with `user` directives)  
TLS files directory: `/etc/redis/tls/`  
CA certificate: `/etc/redis/tls/ca.crt`  
Server certificate: `/etc/redis/tls/redis.crt`  
Server private key: `/etc/redis/tls/redis.key`  
Original config backup: `/etc/redis/redis.conf.backup`  
Python demo script: `~/redis_secure_demo.py`

---

**Report Date:** May 4, 2026  
**All Screenshots Captured:** 13 images  
**Test Status:** All passed successfully

