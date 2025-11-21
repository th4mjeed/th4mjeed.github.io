---

title: "Setting Up Samba as an Active Directory Domain Controller on Linux"
date: 2025-11-19
draft: true
------------


In this guide, we walk through the process of setting up a Samba Active Directory Domain Controller. The goal is to create a fully functional AD environment with the ability to join Windows and Linux clients to the domain.

This setup mirrors how Microsoft Active Directory works, but fully powered by open‑source software.

We'll be using Fedora Linux to configure Samba AD, but you can use any Linux

---

## Preparing the Domain Controller System

We begin by configuring Fedora 43 to act as a domain controller. The hostname must reflect the fully qualified domain name of the AD server.

Set the hostname:

```bash
hostnamectl set-hostname dc1.internal.lan
```

Add the server address to `/etc/hosts` so DNS lookups resolve locally during installation:

```bash
10.10.40.90   dc1.internal.lan dc1
```

Next, update packages and install the Samba DC components along with Kerberos and DNS utilities:

```bash
sudo dnf update -y
sudo dnf install -y samba samba-dc samba-dns samba-winbind krb5-workstation bind-utils
```

Fedora uses systemd‑resolved by default, which conflicts with Samba’s internal DNS. We disable it and ensure Samba will handle DNS:

```bash
sudo systemctl disable --now systemd-resolved
sudo rm -f /etc/resolv.conf
```

Then create a new resolv.conf pointing DNS to the AD server itself:

```bash
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
echo "search internal.lan" | sudo tee -a /etc/resolv.conf
```

At this point, the server is ready for domain provisioning.

---

## Provisioning the Active Directory Domain

Samba includes a provisioning tool that creates the directory structure, Kerberos configuration, and internal DNS zones.

Run provisioning interactively:

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

During setup:

* Realm can be set to **INTERNAL.LAN**
* Domain name can be **INTERNAL**
* Server role must be **dc** (domain controller)
* Select **SAMBA_INTERNAL** DNS backend

When provisioning completes, start the Samba service:

```bash
sudo systemctl enable --now samba
```

To verify that DNS is functioning, check SRV records for Kerberos and LDAP:

```bash
host -t SRV _kerberos._udp.internal.lan
host -t SRV _ldap._tcp.internal.lan
host -t A dc1.internal.lan
```

Confirm that Samba’s internal services are running properly:

```bash
sudo samba-tool processes
```

---

## Verifying Kerberos Authentication

Kerberos is central to Active Directory authentication. We test it using the built‑in Administrator account.

```bash
kinit administrator@INTERNAL.LAN
klist
```

If everything is configured correctly, a Kerberos ticket will be displayed.

---

## Creating Users in Active Directory

To add a test user:

```bash
samba-tool user create employee1
```

This user can later be used to log in to domain‑joined machines.

---

## Joining a Fedora Client to the Domain

On the Fedora client, install the tools required for AD domain membership:

```bash
sudo dnf install -y realmd sssd adcli oddjob oddjob-mkhomedir samba-winbind-clients
```

Discover the domain:

```bash
realm discover internal.lan
```

Join the domain using administrative credentials:

```bash
sudo realm join internal.lan -U administrator
```

Enable automatic home directory creation for domain users:

```bash
sudo pam-auth-update --enable mkhomedir
```

Now, domain accounts can be used to log in through the graphical login screen using:

```
INTERNAL\employee1
```

---

## Joining a Windows 11 Client to the Domain

Windows 11 systems can be joined to the Samba Active Directory domain, enabling centralized authentication just like in a Microsoft AD environment.

### Configure DNS on Windows 11

Before joining the domain, ensure the Windows machine uses the domain controller for DNS resolution:

1. Open Settings → Network & Internet
2. Select the active network adapter
3. Choose Edit DNS settings
4. Set it to Manual and configure:

   * Preferred DNS: 10.10.40.90

Test DNS resolution using Command Prompt:

```
ping dc1.internal.lan
```

### Join Windows to the Domain

1. Open Run (Win + R) → type:

```
sysdm.cpl
```

2. Go to the Computer Name tab
3. Click Change and select Domain
4. Enter:

```
internal.lan
```

5. Provide domain credentials when prompted (such as INTERNAL\administrator)
6. Reboot the system

### Validate Domain Login

After reboot, log in using:

```
INTERNAL\employee1
```

Then verify domain membership:

```powershell
systeminfo | findstr /i domain
```

## Final Notes

With Samba successfully serving as an Active Directory Domain Controller, Linux systems can now authenticate against the domain and Kerberos security is fully operational.

Future enhancements may include:
* Installing RSAT on Windows 11 for easy management of User and Groups
* Configuring Group Policy through Samba tools
* Securing DNS and directory traffic with TLS certificates
* Adding domain file services with Windows‑compatible permissions
* Integrating an AD Certificate Services alternative for Kerberos PKINIT

