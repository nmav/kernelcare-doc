---
sidebarDepth: 1
---

# ePortal

KernelCare Enterprise includes a web management console (ePortal)
for protected environments.

ePortal software can help protect servers located behind the
firewall (i.e. with no internet access) and can be installed both
on-premises and in the cloud.

![Overview](/images/eportal-overview.svg)


## ePortal Hardware Requirements

**Disk**

* 100 GB minimum, 200 GB recommended (20GB in case of cache mode)
* SSD with at least 100 IOPS

:::tip Note
SSD based storage is a crucial requirement for ePortal server.
:::

**CPU**

As far as other requirements concerned, we have tested the following configuration
and the following number of connected servers:

* 10k of connected machines is the maximum for the following requirements:
  * VM
  * 1 VCPU
  * 1 GB RAM

* 75k of connected machines is the maximum for the following requirements:
  * Core i5
  * 1 CPU
  * 4 GB RAM


## Installation

ePortal is compatible with 64-bit versions of EL7/8 based distros like CentOS 7/8, AlmaLinux 8 and
Ubuntu 20.04.

### RHEL-based distros

For installation and workability of ePortal the Nginx web server is required.
We recommend to use stable version from the official Nginx repository:

```
cat > /etc/yum.repos.d/nginx.repo <<EOL
[nginx]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/\$releasever/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOL
```

You can find more information at [https://nginx.org/en/linux_packages.html#stable](https://nginx.org/en/linux_packages.html#stable)

Setup ePortal repo:

```
cat > /etc/yum.repos.d/kcare-eportal.repo <<EOL
[kcare-eportal]
name=KernelCare ePortal
baseurl=https://www.repo.cloudlinux.com/kcare-eportal/\$releasever/\$basearch/
enabled=1
gpgkey=https://repo.cloudlinux.com/kernelcare/RPM-GPG-KEY-KernelCare
gpgcheck=1
EOL
```

Install ePortal:

```
yum install -y kcare-eportal
```

### Ubuntu 20.04

Setup ePortal repo:

```
apt-get install -y --no-install-recommends curl ca-certificates

curl https://repo.cloudlinux.com/kernelcare/kernelcare.gpg -o /usr/share/keyrings/kcare-eportal.gpg

cat > /etc/apt/sources.list.d/kcare-eportal.list <<EOL
deb [signed-by=/usr/share/keyrings/kcare-eportal.gpg] https://repo.cloudlinux.com/kcare-eportal/20.04 focal main
EOL
```

Install ePortal:

```
apt-get update && apt-get install -y --no-install-recommends kcare-eportal
```

## Cache mode

Cache mode allows to greatly reduce disk usage requirements and speed up
initial downloading step.

In cache mode ePortal downloads only lightweight meta information about
patchsets and proxies patch requests from KernelCare agent to the patch server
(Patch Source). Downloaded patch binaries are cached for 2 weeks and
accessible for following requests directly from ePortal and doesn't consume
public internet bandwidth.

![Cache mode](/images/eportal-cache-mode.svg)

### Enable cache mode

1. add

   ```
   CACHE_MODE = True
   ```

   setting into [ePortal config file](#config-files)

2. restart ePortal

   ```
   systemctl restart eportal
   ```

For existing installation you have to refetch meta info for existing patchsets:

```
kcare/kc.eportal cache-mode --fetch-meta
```

### Disable cache mode

1. remove or comment line with

   ```
   CACHE_MODE = True
   ```

   in [ePortal config file](#config-files)

2. download missing patchests

   ```
   kcare/kc.eportal kcare download-missing
   ```

3. restart ePortal

   ```
   systemctl restart eportal
   ```


## Proxy configuration

ePortal can fetch packages and patches via customer's proxy server.

![Proxy](/images/eportal-with-proxy.svg)

On the ePortal machine, you should define the same proxy settings as you use
in the command line.

To do so, add `PROXY = 'http://example.com:3128'`
into [ePortal config file](#config-files):

```
echo -e "\nPROXY = 'http://example.com:3128'" >> /etc/eportal/config
```

ePortal supports SOCKS5 proxy via `socks5://` proxy scheme:

```
PROXY = 'socks5://example.com:1080'
```

Restart ePortal (see [Stopping & Starting](#stopping-starting)
section, choose a corresponding OS).


## Auth configuration

| | |
|-|-|
| `AUTH_PASSWORD_MIN_LENGTH` | Minimal password length, default 5 |
| `AUTH_SESSION_LIFETIME`    | Session lifetime in seconds, by default session will last till browser closing |
| `AUTH_REFRESH_SESSION`     | If `False` (default), expires session after lifetime seconds after login. If `True`, expires session after lifetime seconds of inactivity. |

You can set configuration in [ePortal config file](#config-files).


## Managing Users

You can manage portal using `kc.eportal user` utility:

```
usage: kc.eportal user [-h] [-a] [-c] [-d] [-p PASSWORD] [-r {admin,readonly}] [--note NOTE] [user]

list available users by default

positional arguments:
  user                  user name

optional arguments:
  -h, --help            show this help message and exit
  -a                    action: add new user
  -c                    action: change user
  -d                    action: delete user
  -p PASSWORD, --password PASSWORD
  -r {admin,readonly}, --role {admin,readonly}
  --note NOTE           description
```

To add a user:

```
kc.eportal user -a -p AdminPassword admin
```

To change user's password:

```
kc.eportal user -c -p NewPassword admin
```

Set user as read-only:

```
kc.eportal user -c -r readonly admin
```


## Authentication using LDAP

ePortal supports LDAP authentication via bind operation. Every user should
have a DN with an ability to perform bind request. ePortal doesn't sync
users so you don't need to set special DN/password to access LDAP server.


### Configuration

LDAP configuration settings are available inside `Settings / LDAP` navigation item.

#### URL

Specifies host, port, and protocol of LDAP server to connect to.

Format: `<scheme>://<host>[:port][?params]`.

* **scheme**:
  * `ldap`: a verified encrypted TLS connection, you can specify `tls=0` to use
    insecure connection;
  * `ldaps`: an SSL connection;
* **host**: hostname or ip of your LDAP server;
* **port**: custom port, by default 389 for `ldap` and 636 for `ldaps` scheme;
* **params**:
  * `strict_check=0`: enable/disable strict SSL certificate verification (enabled by default);
  * `tls=0`: enable/disable TLS handshake on connect (enabled by default);
  * `timeout=30`: LDAP server connection timeout (in seconds, 5 by default).

For example:

* `ldap://ldap.forumsys.com?tls=0&timeout=30`: use insecure connection with 30s
  connection timeout.

#### Connection String (DN template)

Connection string is a DN template used in LDAP bind operation. All `%s`
template placeholders will be replaced with username from login form.

For example: `uid=%s,dc=example,dc=com`.

Some LDAP implementations support additional templates for bind requests:

* Active Directory accepts `%s@example.com` template;
* `cn=%s,dc=example,dc=com` or `email=%s,dc=example,dc=com` can be also valid.

#### Filter

Filter is an optional setting which specifies parameters for an additional LDAP
search request to limit users who have access to ePortal. Usually it is a check
of a group user belongs to. If search returns non-empty result user will be
considered authenticated.

Format: `<base>??<scope>?<filter>`, is a part of
[The LDAP URL Format RFC](https://tools.ietf.org/html/rfc2255).

* **base**: a search base, a root node to perform search from;
* **scope**: a search scope;
  * `base`: search only for base (default);
  * `one`: search for direct ascendants only (one level deep from base);
  * `sub`: search in whole tree;
* **filter**: search filter to apply.

For example: `dc=example,dc=com??one?(&(ou=mathematicians)(uniqueMember=uid=%s,dc=example,dc=com))`

Take note. You can use `%s` placeholder in `Filter` setting as well. It will be
replaced with username from login form.

### Login process

When a user logs in to ePortal, ePortal checks the user credentials in the local
database at first. If the credentials are found, the user will be authenticated,
and only if the user credentials are not found, ePortal redirects to LDAP.

When you log in to ePortal with your LDAP credentials for the first time,
a user with LDAP username, read-only permissions, and `LDAP` description
is created in the database by default (`https://<eportal>/admin/user/`).

In the picture, you can see one user created with `kc.eportal` command line
interface and two users logged in with LDAP credentials.

![](/images/eportalLDAPusers.png)

:::tip Note
Take note. All fresh LDAP users have read-only permissions. You can set
admin rights on `User edit` page by turning off `Readonly` field.
:::


### Basic setup example

This and following example are based on public test LDAP server
[ldap.forumsys.com](https://www.forumsys.com/tutorials/integration-how-to/ldap/online-ldap-test-server/).

Here is a list of our users:

```
~$ ldapsearch -H ldap://ldap.forumsys.com -LLL -x -b 'dc=example,dc=com' '(objectClass=person)' dn
dn: uid=tesla,dc=example,dc=com
dn: uid=gauss,dc=example,dc=com
...
```

Basic setup only checks if provided username/password can bind to your directory
service. Take note: all fresh users have read-only permission. So you can
safely allow to login into eportal for all LDAP users and set correct
permissions individually.

For basic setup you should fill `Url` and  `Connection string` settings:

|Setting|Value|
|-|-|
|URL|ldap://ldap.forumsys.com?tls=0|
|Connection string|uid=%s,dc=example,dc=com|

After that you can login using `gauss:password` or `tesla:password` credentials.

### Filter setup example

Filter allows to limit a set of users permitted to login into ePortal, for
example users belonging to `mathematicians` group.

Here are our example groups:

```
~$ ldapsearch -H ldap://ldap.forumsys.com -LLL -x -b 'dc=example,dc=com' '(objectClass=groupOfUniqueNames)' dn
dn: ou=mathematicians,dc=example,dc=com
dn: ou=scientists,dc=example,dc=com
...
```

And `mathematicians` group attributes:

```
~$ ldapsearch -H ldap://ldap.forumsys.com -LLL -x -b 'dc=example,dc=com' '(ou=mathematicians)'
dn: ou=mathematicians,dc=example,dc=com
uniqueMember: uid=euclid,dc=example,dc=com
uniqueMember: uid=riemann,dc=example,dc=com
uniqueMember: uid=euler,dc=example,dc=com
uniqueMember: uid=gauss,dc=example,dc=com
uniqueMember: uid=test,dc=example,dc=com
ou: mathematicians
cn: Mathematicians
objectClass: groupOfUniqueNames
objectClass: top
```

ePortal LDAP settings to limit allowed users:

|Setting|Value|
|-|-|
|URL|ldap://ldap.forumsys.com?tls=0|
|Filter|dc=example,dc=com??one?(&(ou=mathematicians)(uniqueMember=uid=%s,dc=example,dc=com))|
|Connection string|uid=%s,dc=example,dc=com|

Filter idea is to find the `mathematicians` group which has an attribute
`uniqueMember` equal to DN of our user who tries to login.

After that you can login using `gauss:password` but not `tesla:password` credentials.


## Accessing ePortal

To access KernelCare.eportal management console, connect to:
**http://YOUR_IP/admin**

And enter your login & password

![](/images/access_eportal.png)

You can manage your login information using [kc.eportal tool](#managing-users).


## PatchSet deployment

ePortal has built-in mechanism to download latest patches. To start
using it, click _Settings_ link in ePortal navigational bar and
then click _Patch Source_. To get access, please contact
[sales@tuxcare.com](mailto:sales@tuxcare.com) and request your ePortal
access credentials.

![](/images/eportal-edit-patch-source.png)

You can choose only a limited set of distros to fetch to reduce disk
space and network bandwidth requirements.

Once you set up patch source access info, you will get to a list of available patchsets.

![](/images/eportal-dashboard.png)

* **Usage**: amount of servers using particular patchset on any
  feed. You can click on the number to see a list of servers using the patchset.

* **Patchsets**: name of the patchset. Click on a name will show patchset
  changelog.

* **Feed name**: cells show patchset deployment and server usage status on particular feed.
  Numbers are also clickable to filter server list.

* **Manage**: opens dialog to enable/disable the patchset.

### Manage patchset page

![](/images/eportal-manage.png)

In this dialog you can control which patchsets will be enabled/disabled on
corresponding feeds.

* **Enable all up to this patchset**: action will enable all patchsets
(excluding **disabled**) up to this patchset.

* **Enable**: action will enable only current patchset.

* **Disable**: action will disable only current patchset. Disabled patchsets
are immune to auto-update and mass-enabling via **Enable all up to this patchset**
action.

To unroll patchset from CLI, run:

```
kc.eportal --unroll 16012017_1
```

### Script to automatically install latest patchsets

It determines if latest patches are available and installs them to the **default feed**.

```
kc.eportal --get-latest
```

If you need to update other feeds that are marked for auto update instead of
the default one, use a different argument:

```
kc.eportal --auto-update
```

Note, that the latter command will update only feeds that has `Autoupdate`
attribute.

### Script to deploy release from a file

If you have no opportunity to connect your ePortal server to the KernelCare
patch server to download patchsets directly from it, you can do it manually.

If you have a location with already downloaded patchsets, and would like to
identify the latest patchset file to be moved, you can compare the lists of
archives you have with the content of the `/usr/share/kcare-eportal/arch/`
folder.

After that, upload the selected patchsets to your ePortal server and run the
`kc.eportal --deploy` command for each of them.

#### Example

Lets find out the difference between a test and a production ePortal instances:

```bash
$ comm -23 \
    <(ssh eportal-test "ls /usr/share/kcare-eportal/arch/K*.tar.bz2" | sort -h) \
    <(ssh eportal-prod "ls /usr/share/kcare-eportal/arch/K*.tar.bz2" | sort -h) | tee patchsets.diff
```

Upload patchsets to the production:

```bash
$ cat patchsets.diff | xargs -Phav {} rsync -iv eportal-test:{} /tmp/
$ rsync -Phav /tmp/K*.tar.bz2 eportal-prod:/tmp/
```

Run the deployment tool with patchset file name as a parameter:

```bash
$ ssh eportal-prod 'ls /tmp/K*.tar.bz2 | sort -h | xargs -n1 kc.eportal kcare deploy'
2021-12-02 01:25:06,555 - eportal.patches - INFO - K04082020_1 was enabled in main
...
```

:::danger Note
Please note that procedure above should be done for all other types of
patchsets (like libcare and qemu) *separately*. Use corresponding file prefixes
and commands like `kc.eportal libcare deploy`.
:::

### Clean obsolete releases

The list of provided patches is reduced once in a month (old patches that have
newer versions are removed), but resources are not deleted from ePortal machine.
The control of resource deletion is in charge of the ePortal administrator. To
free disk space on ePortal old release resources could be removed.

To check what releases will be removed:

```
DRY_RUN=1 kc.eportal --clean-releases
```

To remove obsolete releases:

```
kc.eportal --clean-releases
```

## LibCare PatchSet Deployment

Starting from version 1.18, ePortal supports the LibCare patchset management. It
is accessible from the `Patches / LibCare` navigation item. LibCare patches
use the same Patch Source credentials, and you don't need to perform additional
configuration.

![](/images/eportal-libcare-feed.png)


### CLI to install the latest patchsets

To update the default feed, run the following command:

```
kc.eportal libcare update
```

To update the `test` feed, run the following command:

```
kc.eportal libcare update --feed test
```

To update all auto-feeds, run the following command:

```
kc.eportal libcare auto-update
```

### CLI to deploy patchset from archive

```
~$ kc.eportal libcare deploy --help
usage: kc.eportal libcare deploy [-h] [--feed FEED] [--disabled] archive

positional arguments:
  archive      path to archive

optional arguments:
  -h, --help   show this help message and exit
  --feed FEED  feed to deploy archive to
  --disabled   do not enable patchset after deploy
```

For example:

```
kc.eportal libcare deploy --feed test /tmp/U20210129_02.tar.bz2
```

This command will deploy and enable the `U20210129_02` patchset in to the `test` feed.

## QEMU PatchSet Deployment

Starting from version 1.25, ePortal supports the QEMU patchset management. It
is accessible from the `Patches / QEMUcare` navigation item. QEMU patches
use the same Patch Source credentials, and you don't need to perform additional
configuration.

![](/images/eportal-qemu-feed.png)


### CLI to install the latest patchsets

To update the default feed, run the following command:

```
kc.eportal qemu update
```

To update the `test` feed, run the following command:

```
kc.eportal qemu update --feed test
```

To update all auto-feeds, run the following command:

```
kc.eportal qemu auto-update
```

### CLI to deploy patchset from archive

```
~$ kc.eportal qemu deploy --help
usage: kc.eportal qemu deploy [-h] [--feed FEED] [--disabled] archive

positional arguments:
  archive      path to archive

optional arguments:
  -h, --help   show this help message and exit
  --feed FEED  feed to deploy archive to
  --disabled   do not enable patchset after deploy
```

For example:

```
kc.eportal qemu deploy --feed test /tmp/U20210818_01-qemu.tar.bz2
```

This command will deploy and enable the `U20210818_01-qemu` patchset in to the `test` feed.

## Managing Keys

To register new servers you need to create a KernelCare key that will be used
for server registration. To go to the list of keys, click the _Keys_ navigation
item.

![](/images/key-menu_zoom70.png)

* To edit a key, click ![](/images/eportal_keys_edit.png). The _Edit_ tab opens.
* To remove a key, click ![](/images/eportal_keys_remove.png). Please note, that
  removing the key would remove all servers under that key.
* Click a key to go to Servers tab with the list of
  [servers registered](#managing-servers) under that key.
  You can also remove servers on that tab.

To create a new registration click _Create_ tab.

   ![](/images/key-creation_zoom70.png)

Fill in the following fields:

* **Key** —  you can provide a key value or leave the field empty, so an
  automatically generated key will be used
* **Description** — you can provide a description for the key
* **Server Limit** — the amount of servers that can be registered under that key
* **Feed** — select a specific feed or leave empty.

Click _Save_ to add the key. The new registration key will be created and added
to the list. The key itself will be used as a part of the registration command
on an individual server.

Click _Save and Add Another_ to save this key and add one more key.

Click _Save and Continue Editing_ to add the key and open the key edit tab.

Click _Cancel_ to return to the key list tab without adding a new key.

### CLI to manage keys

```
~$ kc.eportal key --help
usage: kc.eportal key [-h] [-a] [-c] [--note NOTE]
                      [--server-limit SERVER_LIMIT] [--feed FEED]
                      [key]

list available keys by default

positional arguments:
  key                   key name

optional arguments:
  -h, --help            show this help message and exit
  -a                    action: add new key
  -c                    action: change key
  --note NOTE           key description
  --server-limit SERVER_LIMIT
                        maximum number of servers allowed
  --feed FEED           feed to associate key to
```

List keys:

```
kc.eportal key
```

Autogenerate key:

```
kc.eportal key -a
```

Create key with specified value:

```
kc.eportal key -a test
```

Set feed to key:

```
kc.eportal key -c test --feed test
```


## Managing Servers

You can see servers belonging to the key by clicking on the key itself in
[Managing Keys](#managing-keys) interface.

![](/images/server_list_1_zoom70.png)

The screen shows servers registered under the key, their IP, hostname,
effective kernel as well as the time of registration and last check in.

To view the list of all servers IDs that are connected to the particular key,
do the following:

* In the UI go to the page with the list of keys. Then click the particular
  key. The list of servers connected to this key will be displayed.

To view the list of all servers IDs that are not connected to any key, use the
<span class="notranslate">_Servers_</span> button on the navigation bar.

![](/images/eportal-servers.png)

Also you can use CLI to see pairs of key/number of servers run:

```
$ kc.eportal --list-servers
Count | Key
----- + --------------------------------
    0 | 2shcolu7Y1x6885Q
    2 | 6J89aS44j6OmTr05
```

### Clean inactive servers

Servers inactive for more than 30 days can be removed using the followed command:

```
$ kc.eportal server clean --inactive
2 servers were deleted
```

Clean-up of inactive servers can be performed automatically every day or rarer.
To enable daily auto clean-up, you can add an option `CLEAN_INACTIVE_SERVERS_PERIOD = 3600 * 24` (value in seconds)
in the [ePortal config file](#config-files).

### Show extended check-in statistics in admin UI

A new table is added to the starting page. This table displays the following:
* Total number of servers.
* Number of servers that checked in for the past 48 hours.

The number of servers for each key is listed on the _Keys_ page.


## Feed Management

Feeds are intended to manage patchsets on the server, and they provide
a possibility to bind a set of patches to a specific key. Possible use cases:
for preliminary testing of patches, for applying updates to groups of
servers with the similar hardware, etc.

To get into Feeds Management interface go to Settings → Feeds:

![](/images/feed-button_zoom70.png)

On this page a user can manage the existing feeds: create, delete, edit.

![](/images/feed-menu_zoom70.png)

Available options:
* Name — a name of a feed.
* Auto update — enable and disable automatic downloading of patches to this feed.
* Deploy after X hours — a delay in hours between the moment the patchset is
  available for deployment and the moment it is installed to the feed.

Every 10 minutes ePortal checks for new patches on the main patch server. If
a new patch is available, it is uploaded to the ePortal server. Note: it is
uploaded but is not deployed. The patch availability time is considered
starting from the moment a new patch appears on the ePortal, and that time is
taken into account in `Deploy after X hours` option. So, if a user sets `Deploy
after X hours = 10`, the patch will be deployed to the feed 10 hours after it
has been downloaded to the ePortal server.

To make the feed auto-update immediately, so new patches are loaded to
the feed immediately after they are available on ePortal, set `Deploy after
X hours = 0`.

A special case is a clean installation when ePortal is installed on a new
server (there aren't any downloaded archives with patches and feeds with
deployed patchsets, including default feed). In this case, if a user
creates a new feed and sets Deployed after X hours option right away,
then all patches (from the oldest to the latest available) will be
deployed to the feed after the specified X hours. This is because the
archives are downloaded from scratch and will be considered as “just
appeared on ePortal” — that is, all patches will have the same
appearance time on ePortal from which the option Deploy after X hours
will repel.

![](/images/feedmanagement3_zoom70.png)

On the main ePortal page, a user can set the corresponding key <> feed pair.
This is done in the key creation interface or when editing a key.

![](/images/feedmanagement4_zoom70.png)

By default, a new key is bound to the default feed, alternatively, a user can
choose a desired feed from the drop-down menu.

![](/images/feedmanagement5_01_zoom70.png)

:::tip Note
When removing a feed all keys attached to this feed will be moved to the
default feed.
:::

![](/images/feedmanagement6_zoom70.png)

### CLI to manage feeds

```
~$ kc.eportal feed --help
usage: kc.eportal feed [-h] [-a] [-c] [--auto] [--no-auto]
                       [--deploy-after hours]
                       [feed]

list available feeds by default

positional arguments:
  feed                  feed name

optional arguments:
  -h, --help            show this help message and exit
  -a                    action: add new feed
  -c                    action: change feed
  --auto                update feed automatically
  --no-auto             don't update feed automatically
  --deploy-after hours  deploy after specified hours
```

List feeds:

```
kc.eportal feed
```

Add auto updated feed:

```
kc.eportal feed -a test --auto
```

Change feed into delayed feed:

```
kc.eportal feed -c test --deploy-after 12
```


## Adding extra Tag field
 
To add an extra Tag field for the server, run:

```
kcarectl --tag command
```

where `command` is a parameter defined by a user. This parameter will be
displayed in UI for the server. User could add multiple tags for each server.
Each tag should be separated with `;` symbol.

Example:

```
kcarectl --tag "env:prod;ubuntu"
```

This server has two tags : `env:prod` and `ubuntu`.

`env:prod` is a parameter that has tag name `env` and the value `prod`.

![](/images/addingextratagfield_zoom88.png)

To remove all tags from a particular server, run:

```
kcarectl --tag ""
```

Where `""` is a parameter to delete the previously defined tag.


## ePortal API

You may need to create a separate ePortal account for API usage with:

```
kc.eportal -a api-user -p <password>
```

It can be useful for SSO/LDAP setups to be able to pass HTTP API credentials via
basic auth.


### GET /admin/api/servers

Filters servers on various criteria, iterate through list and get
server counts.

Requires basic authorization with read only user permissions.

**Query string parameters:**

* `hostname`: String, optional. Return servers with a hostname like value. Value can
  contain `%` placeholder to match any string.
* `ip`: String, optional. Return servers with IP like value. Value can
  contain `%` placeholder to match any string.
* `feed`: String, optional. Return servers attached to feed. Use `main`
  for default feed.
* `key`: String, optional. Return servers registered to key.
* `registered_age`: Integer, optional. Return servers registered more than
  `registered_age` days ago. If `registered_age` is negative then return
  servers registered less or equal -`registered_age` days ago. For example:
  `registered_age=-3` means to return servers registered 3 or less days ago.
  Hours can be specified by adding `h` suffix: `registered_age=4h` selects
  servers registered more than 4 hours ago.
* `checkin_age`: Integer, optional. Return servers sent check-in more than
  `checkin_age` days ago. For negative values see `registered_age`.
* `updated_age`: Integer, optional. Return servers loaded patches more than
  `updated_age` days ago. For negative values see `registered_age`.
* `is_updated`: Boolean, optional. If `true` then return servers updated to latest
  available patches. If `false` return servers without latest patches.
* `limit`: Integer, optional. Limit result to a specified number of entries. By
  default `limit` is 10.
* `offset`: Integer, optional. Set result to a specified offset. `limit` and
  `offset` can be used to iterate through result.
* `only_count`: Boolean, optional. Return server count only.
* `tag`: String, optional. Adds filter by server tag. E.g `tag=env:test` or `tag=ubuntu`.

**Response:**

```json
{
    "count": 42,
    "limit": 10,
    "offset": 0,
    "result": [
        {
            "id": "99c97tz44uKX13b5",
            "ip": "10.51.16.72",
            "hostname": "localhost.localdomain",
            "key": "some-key",
            "feed": "main",
            "registered": "2021-04-08 16:43:23.907671",
            "checkin": "2021-07-12 17:35:56.065077",
            "updated": "2021-06-01 16:37:03.000000",
            "euname": "3.10.0-1160.25.1.el7",        // effective kernel version
            "release": "3.10.0-957.5.1.el7.x86_64",  // installed kernel version
            "kernel_id": "9647204d2708cad906a75944ee56ac68fc5b5704",
            "patch_level": 49,
            "patch_type": "default",
            "tags": null,
            "uptime": 6394092,
            "version": "#1 SMP Fri Feb 1 14:54:57 UTC 2019",
            "virt": "kvm",
            "kcare_version": "2.44-2",
            "distro": "CentOS Linux",
            "distro_version": "7.6.1810",
            "machine": "x86_64",
            "processor": "x86_64"
        },
        ...
    ]
}
```

### POST /admin/api/delete_server

Removes registered servers by IP or hostname

Requires basic authorization with admin user permissions.

**Query string parameters:**

* `hostname`: String, optional. Server's hostname to delete.
* `ip`: String, optional. Server's IP to delete.

Endpoint requires at least one parameter `hostname` or `ip`.

**Response:**

Response contains number of deleted servers.

```json
{
    "result": 1
}
```

For example:

```
curl --user admin:admin-password -X POST http://your-eportal-domain/admin/api/delete_server?ip=192.168.1.1
```

Below are examples of using `/admin/api/delete_server` endpoint with various
CM products. Take note, you need a host IP or a hostname to delete it via API.
It can be more convenient to use on of:

* if you have an access to a host (it is alive) it's a way more simpler to call `kcarectl --unregister` to delete the host.
* if host is already destroyed you can consider using bulk [inactive server cleanup](#clean-inactive-servers).


#### Ansible playbook

Integrating ePortal API access with Ansible is now possible. Rather than calling the kernelcare agent to perform tasks, it is possible to replace it with direct calls to ePortal to achieve the same results.

Unregister KernelCare agent through API playbook:

```json
- hosts: kernelcare
  vars:
    eportal_srv: http://eportal.address.here[:port if needed]

  tasks:
    - name: unregister kernelcare agent through API
      uri:
        url: "{{ eportal_srv }}/admin/api/delete_server?ip={{ ansible_default_ipv4.address|default(ansible_all_ipv4_addresses[0]) }}"
        method: POST
        user: your-api-user
        password: your-api-user-password
        force_basic_auth: yes
```

Example:

```json
- hosts: kernelcare
  vars:
    eportal_srv: http://192.168.246.110

  tasks:
    - name: unregister kernelcare agent through API
      uri:
        url: "{{ eportal_srv }}/admin/api/delete_server?ip={{ ansible_default_ipv4.address|default(ansible_all_ipv4_addresses[0]) }}"
        method: POST
        user: api-user
        password: dummy
        force_basic_auth: yes
```

Ad hoc run with:

```
ansible-playbook -u ansible  ./kernelcare.yml
```

This can be called during machine tear down to properly remove the server from ePortal.

#### Chef recipe

Having tighter integration with automation tools like Chef has always been a goal for KernelCare and related tools like ePortal. The following recipe demonstrates how to use the delete_server api to correctly remove a server from ePortal during tear down, and can be integrated with your other recipes to avoid manual operations.

Unregister KernelCare agent through API recipe:

```json
eportal_url = "EPORTAL URL"
eportal_user = "EPORTAL API USER NAME"
eportal_password = "EPORTAL API USER PASSWORD"

# the ip to unregister can be pulled automatically from the system where the recipe is applied, or specified manually (by replacing the following line with a simple assignment)
ip_to_unregister = "#{node['network']['interfaces'][node['network']['default_interface']]['addresses'].select{|k,v| v['family'] == "inet"}.keys.first}"


http_request "kernelcare-unregister-api" do
  url "#{eportal_url}/admin/api/delete_server?ip=#{ip_to_unregister}"
  action :post
  headers({'AUTHORIZATION' => "Basic #{
    Base64.encode64("#{eportal_user}:#{eportal_password}")}",
  })
end
```

Example (kernelcare-unregister-api.rb):

```json
eportal_url = "http://192.168.246.110"
eportal_user = "api-user"
eportal_password = "dummy"

ip_to_unregister = "#{node['network']['interfaces'][node['network']['default_interface']]['addresses'].select{|k,v| v['family'] == "inet"}.keys.first}"

http_request "kernelcare-unregister-api" do
  url "#{eportal_url}/admin/api/delete_server?ip=#{ip_to_unregister}"
  action :post
  headers({'AUTHORIZATION' => "Basic #{
    Base64.encode64("#{eportal_user}:#{eportal_password}")}",
  })
end
```

Ad hoc run with:

```
chef-apply kernelcare-unregister-api.rb
```

Output:

```json
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * http_request[kernelcare-unregister-api] action post
    - http_request[kernelcare-unregister-api] POST to http://192.168.246.110/admin/api/delete_server?ip=192.168.246.40
```

This can be called during machine tear down to properly remove the server from ePortal.

#### Puppet task

Puppet provides the framework to run tasks on target systems. The following is a bash script that can run as such a task and demonstrates how to use the delete_server api to correctly remove a server from ePortal during tear down. It can be integrated with your other removal scripts or tasks to avoid manual operations.

Unregister KernelCare agent through API call:

```bash
#!/bin/bash

EPORTAL_API_USERNAME=<your ePortal api user name>
EPORTAL_API_PASSWORD=<your ePortal api user password>
EPORTAL_URL='your ePortal URL'

#this is taken from the primary ip in the system. If awk is available, it is used, but a fallback using other common tools is also provided
if hash awk 2>/dev/null; then
        IP_TO_UNREGISTER=`ip route get 1 | awk '{print $(NF-2);exit}'`     # using awk
else
        IP_TO_UNREGISTER=`ip route get 1 | cut -f 3 -d" "| head -1`        # simpler alternative for when awk is not available
fi

curl -kL -u "${EPORTAL_API_PASSWORD}"':'"${EPORTAL_API_PASSWORD}" -X POST "${EPORTAL_URL}"'/admin/api/delete_server?ip='"${IP_TO_UNREGISTER}"
```

Example (unregister_server.sh):

```bash
#!/bin/bash

EPORTAL_API_USERNAME=admin
EPORTAL_API_PASSWORD=admin
EPORTAL_URL='http://192.168.246.110'

#this is taken from the primary ip in the system. If you want to pick a different one, adjust the next line.
if hash awk 2>/dev/null; then
        IP_TO_UNREGISTER=`ip route get 1 | awk '{print $(NF-2);exit}'`     # using awk
else
        IP_TO_UNREGISTER=`ip route get 1 | cut -f 3 -d" "| head -1`        # simpler alternative for when awk is not available
fi

curl -kL -u "${EPORTAL_API_PASSWORD}"':'"${EPORTAL_API_PASSWORD}" -X POST "${EPORTAL_URL}"'/admin/api/delete_server?ip='"${IP_TO_UNREGISTER}"
```

#### Puppet Plan

If you prefer to have a plan rather than a task, then you can create one from this script with the following steps:

* Create a new directory called `eportal_puppet`
* Inside this directory, create a bolt project:
  ```
  bolt project init
  ```
* Create a scripts directory inside it
* Place the script above inside of it (call it `unregister_server.sh`)
* Create the bolt plan using:
  ```
  bolt plan new eportal_puppet::unregister_server --script eportal_puppet/scripts/unregister_server.sh
  ```
* Now your plan is ready and can be called directly with:
  ```
  bolt plan run eportal_puppet:unregister_server -t <TARGETS>
  ```

Ad hoc run example with:

```
bolt plan run eportal_puppet::unregister_server -t 192.168.246.110
```

This can be called during machine tear down to properly remove the server from ePortal.


## How to setup ePortal to use HTTPS

Some assumptions for a server where e-portal is deployed:

### Terminate TLS on ePortal public nginx web server

1. A firewall allows connections to 443 port.
2. Private and public keys are downloaded on the server.

* Edit SSL configuration template according to your certificates:

```
mv /etc/nginx/eportal.ssl.conf.example /etc/nginx/eportal.ssl.conf
vi /etc/nginx/eportal.ssl.conf
```

* Include this configuration into the main one:

```
sed -e '3iinclude eportal.ssl.conf;' -i /etc/nginx/conf.d/eportal.conf
```

* Restart nginx:

```
service nginx restart
```

### Terminate TLS on external balancer

You can use any reverse proxy server of choice to terminate TLS traffic and
forward requests to ePortal instance. The only requirement is to provide correct
`Host` and `X-Forwarded-Proto` headers.

For example balancer nginx config:

```
server {
   listen       443 ssl;
   server_name  your-eportal-domain;

   # ssl stuff ...
   # ssl_certificate  ...
   # ssl_certificate_key ...

   location / {
      proxy_pass http://eportal-instance-address;
      proxy_set_header Host $http_host;
      proxy_set_header X-Forwarded-Proto https;
   }
}
```

### Update agent configuration

In order to communicate with e-portal, updated to https, you need to modify
KernelCare configuration file on all the servers if they have IPs hard coded servers
settings.

To do that, update `PATCH_SERVER` and `REGISTRATION_URL` options:

```
vi /etc/sysconfig/kcare/kcare.conf
```

So, after editing your `/etc/sysconfig/kcare/kcare.conf` should contain updated
`PATCH_SERVER` and `REGISTRATION_URL` environment variables like in the example
below:

```
PATCH_SERVER=https://eportal_domain_name/
REGISTRATION_URL=https://eportal_domain_name/admin/api/kcare
```

As an alternative you can use ePortal's special endpoint to update patch server address:

```
curl -s https://eportal_domain_name/set-patch-server | bash
```


## High availability

Starting from version 1.28, ePortal supports application level replication. It
allows to propagate changes in both ways – from a leader to followers and from
followers to the leader.

Configuration settings related to replication and can be changed in [ePortal config file](#config-files):

* `NODE_URL`: URL to the self instance, needed for auto-discovery purposes
* `LEADER_URL`: URL to the instance from where to fetch changes
* `REPLICATION_SHARED_KEY`: authorization key to access the replication data
* `REPLICATION_CHECK_CERT`: use strict TLS cert check for replication
  requests. It can be useful to deploy self signed certificates on replicas and
  get encrypted replication traffic without getting proper TLS certificate.

A leader node discovers followers automatically and after that fetches the changes.

You can use any convenient way to balance agent requests to the ePortal
cluster. For example, you can add multiple IP addresses to the DNS name or use
an HTTP balancer.


### Basic setup

1. Prepare two fresh ePortal instances and assign DNS name to IPs. For example:

   * eportal1.corp -> 192.168.1.11
   * eportal2.corp -> 192.168.1.12
   * eportal.corp -> 192.168.1.11, 192.168.1.12

2. Let's choose `eportal1` as a leader and `eportal2` as a follower.
   `eportal.corp` is a cluster hostname to use on servers with KernelCare
   agent.

   :::danger Warning!
   Please do not use a cluster hostname to perform operations with the ePortal admin UI.
   :::

3. On the `eportal1` define `NODE_URL` and `REPLICATION_SHARED_KEY` in the
   configuration file:

   ```
   NODE_URL = 'http://eportal1.corp'
   REPLICATION_SHARED_KEY = 'secret'
   ```

4. On the `eportal2` define `NODE_URL`, `LEADER_URL` and `REPLICATION_SHARED_KEY`:

   ```
   NODE_URL = 'http://eportal2.corp'
   LEADER_URL = 'http://eportal1.corp'
   REPLICATION_SHARED_KEY = 'secret'
   ```

5. That's it. After ePortal restart on both hosts, changes on the `eportal1`
   will be replicated to the `eportal2` instance and vice versa.

:::tip Note
You can change the KernelCare agent settings to point to a cluster hostname via:
```
curl -s http://eportal.corp/set-patch-server | bash
```
:::

### Adding node to an existing ePortal instance

If you already have a working ePortal instance, you can setup a second node,
define `NODE_URL` and `LEADER_URL` on both instances and trigger full sync on
the follower instance:

```
[root@eportal2.corp]$ kc.eportal replication --full-sync
```

### Monitoring

You can use `kc.eportal replication --short-status` CLI command to check replication
status. It exits with code `1` if there are some issues with replication and
outputs JSON:

```json
{
  "ok": true,       // overall health status
  "lag": 1,         // overall replication lag in seconds
  "last_sync": 1,   // overall sync delta in seconds (how many seconds ago was successful communication with an upstream)
  "upstream": {     // per upstream metrics
    "node2": {
      "ok": true
      "lag": 1,
      "last_sync": 1,
    },
    "node3": {
      "ok": true
      "lag": 1,
      "last_sync": 1,
    }
  }
}
```

### Other replication setups

For one leader and two follower setup:

```
     +-> leader <-+
     |            |
     |            |
follower1       follower2
```

there is a chance to lose a `leader` node and `follower1`/`follower2` will not
communicate with each other. To mitigate this issue, you can deploy a ring
replication:

```
  +-> node1 --+
  |           |
  |           v
node2 <---- node3
```

where each instance follows another node.


### Caveats

1. Replication uses HTTP transport. If you deploy ePortal without SSL
   termination, the replication data will be transferred as-is unencrypted.

2. For a round-robin balancer setups (DNS or HTTP balancer round-robin)
   KernelCare agent can run onto replication lag in case of sequential
   registration and following update. You can introduce 10s timeout or repeat
   in your configuration management logic to mitigate the issue.

3. Replication log is kept for 7 days. If a node loses connectivity for more
   than 7 days it skips some changes.


## Deploying KernelCare Enterprise

You can find instructions to install KernelCare agent on http://your-eportal-domain/admin/docs/.

To install agent through ePortal you can use special endpoint `http://your-eportal-domain/install-kernelcare`:

```
$ export KCARE_MAILTO=admin@mycompany.com
$ curl -s http://eportal.mycompany.com/install-kernelcare | bash
$ kcarectl --register my-key
```

Exporting `KCARE_MAILTO` is an optional step, it helps to configure
corresponding entry in `/etc/cron.d/kcare-cron` to receive all notifications
related to failed KernelCare updates.

### KernelCare Enterprise client config file

KernelCare Enterprise client configuration file is located in
`/etc/sysconfig/kcare/kcare.conf`

Example:

```
AUTO_UPDATE=True
PATCH_SERVER=http://10.1.10.115/
REGISTRATION_URL=http://10.1.10.115/admin/api/kcare
```

If `AUTO_UPDATE` set to `True`, KernelCare client will check in every 4 hours,
and try to download and apply latest patches

`PATCH_SERVER` - server from which patches will be downloaded

`REGISTRATION_URL` - URL used to register/unregister server

### Deployment Automation

It is possible to use automation tools like Ansible/Puppet/Chef/Salt to install
and operate KernelCare on a big number of systems.

The deployment process includes:

* Running `http://your-eportal-domain/install-kernelcare` script. You can
  pre-download it (verify checksum) or directly execute
  `curl -s http://your-eportal-domain/install-kernelcare | bash`.
* Registering KernelCare agents using an activation key

#### Ansible

To start the automated deployment, you need to specify the following information:

* ePortal server name (or IP) in the `eportal_srv` Ansible variable. Other config
  file options can be found at [Config Options](/config-options/) and
  [KernelCare client config file](#kernelcare-enterprise-client-config-file)
  (ePortal).

* an activation key in the `activation_key` Ansible variable. Activation keys can
  be generated in ePortal as described in
  [Managing Keys](#managing-keys) (ePortal).

Ansible playbook for deployment phase may look like:

```
- hosts: kernelcare
  vars:
    eportal_srv: http://192.168.250.19
    activation_key: my-key
  tasks:
    - name: Download the installation shell script
      get_url:
        url: "{{ eportal_srv }}/install-kernelcare"
        dest: /root/kc-install.sh
        mode: '0700'

    - name: Run the installation shell script
      command: /root/kc-install.sh

    - name: register KernelCare agents
      command: /usr/bin/kcarectl --register {{ activation_key }}
```

Ansible playbook file example for KernelCare agent removal:

```
- hosts: kernelcare
  tasks:
    - name: unregister KernelCare agents
      command: /usr/bin/kcarectl --unregister
      ignore_errors: yes
      args:
        removes: /usr/bin/kcarectl

    - name: remove kernelcare package
      package:
        name: kernelcare
        state: absent
```

### Changing ePortal IP/domain

You can change ePortal address at any moment by calling two special endpoints on hosts with KernelCare agent:

```
curl -s http://your-eportal-domain/set-kernelcare-repo | bash
curl -s http://your-eportal-domain/set-patch-server | bash
```

* `set-kernelcare-repo` configures new package repo to download agent updates
through ePortal.
* `set-patch-server` configures new address of patch server.


## Migrate ePortal

Here is a procedure to migrate feed/server/key configuration and patches data from one ePortal
to another. ePortal version on a new host should be >=1.18 and equal or greater
than on an old host.

If you migrate from Debian-based to Debian-based system you can simply:

* Stop ePortal on both hosts.
* Copy `/var/lib/eportal` to a new host. Note: directory owner must stay as `eportal:eportal`.
* Copy config `/etc/eportal/config` if it exists.
* Run migration `/usr/share/kcare-eportal/migratedb.py --upgrade` on a new host.
* Start ePortal on a new host.

Migration from RHEL-based distro is more elaborate. Later we refer to a `$BASE_DIR` variable in scripts.
You can export it for RHEL-based distros:

```
export BASE_DIR=/usr/share/kcare-eportal
```

or for Debian-based distros:

```
export BASE_DIR=/var/lib/eportal
```

or for docker-based installation use corresponding data directory. For example if you
run container as `docker run -v /var/lib/eportal/data:/var/lib/eportal/data kernelcare/eportal`
use /var/lib/eportal/data as `$BASE_DIR`:

```
export BASE_DIR=/var/lib/eportal/data
```

* Stop ePortal on both hosts:

  ```
  [old-host ~]# systemctl stop eportal
  ```

  ```
  [new-host ~]# systemctl stop eportal
  ```

* Remove db files on a new host:

  ```
  [new-host ~]# rm $BASE_DIR/*.sqlite*
  ```

* Copy database files `$BASE_DIR/*.sqlite*` from an old host to a new host.

* (Optional) copy runtime LDAP and Feed Source settings `$BASE_DIR/config/config.json`.

* (Optional) copy ePortal web app settings `/usr/share/kcare-eportal/config/local.py`
  or for new ePortal `/etc/eportal/config` to `/etc/eportal/config` on a new
  host if it exists.

* Copy `$BASE_DIR/instance-id` so our billing can know that the new ePortal was
  migrated and to avoid double billing.

* Copy patch data `$BASE_DIR/{arch,resources,user-resources}` to a new host.

* Set file owner. For RHEL-based distros:

  ```
  [new-host ~]# chown -R nginx:nginx $BASE_DIR
  ```

  For Debian-based distros:

  ```
  [new-host ~]# chown -R eportal:eportal $BASE_DIR
  ```

* Run migration `/usr/share/kcare-eportal/migratedb.py --upgrade` on a new host.

* Start ePortal on the new host:

  ```
  [new-host ~]# systemctl start eportal
  ```


## Database backup

### Create backup

kc.eportal utility have an option to create database backup

`kc.eportal backup-db <path_to_directory>` - creates backup in provided directory. If directory is not exists it will be created.

For example:

```
kc.eportal backup-db /usr/share/kcare-eportal/db-backup_$(date '+%Y-%m-%d')
```

### Recover from backup

You can replace existing databases (by default, in eportal home directory) with the backup files.

IMPORTANT: ePortal should be stopped before replacing database files. Backup files should have owner and group `nginx:nginx`.

For example:

```
systemctl stop eportal
cp -f /path/to/backup_dir/* /usr/share/kcare-eportal/
chown nginx:nginx /usr/share/kcare-eportal/*.sqlite*
systemctl start eportal
```

Also, you need to check remote patchsource settings (Login, Password, Download URL in eportal UI), and after it download patches

```
kc.eportal kcare download-missing
kc.eportal libcare download-missing
kc.eportal qemu download-missing
kc.eportal db download-missing
```


## Config files

* ePortal config: `/etc/eportal/config`. Config for old versions (<1.35): `/usr/share/kcare-eportal/local.py`.
* Web Server (nginx) config: `/etc/nginx/conf.d/eportal.conf`.
* Data directory (RHEL based distros): `/usr/share/kcare-eportal`.
* Data directory (Debian based distros): `/var/lib/eportal`.


## Stopping & starting

To stop/start/reload/restart configuration of nginx server:

```
$ systemctl stop|start|reload|restart nginx
```

To stop/start/restart ePortal (Python):

```
$ systemctl stop|start|restart eportal
```


## Log files

ePortal log: `journalctl -u eportal`

Nginx ePortal access log: `/var/log/nginx/access.log`

Nginx error log: `/var/log/nginx/error.log`


### Log rotation

By default there is no predefined parameters for ePortal's logs rotation. If you
want to enable it for the files listed above:

 - Install `logrotate` package
 - Create/edit `/etc/logrotate.d/eportal` configuration file

Example logrotate config:

```
/var/log/nginx/kcare-eportal.log {
    daily
    rotate 5 # keep 5 last archives
    missingok # it's ok if there is no such file
    notifempty # do nothing if file is empty
    compress
}
```


## Nagios & Zabbix support

KernelCare.ePortal since version 1.2 supports server monitoring similar to
[Nagios](/kc-agent-monitoring/#nagios-plugin)
& [Zabbix](/kc-agent-monitoring/#zabbix-template) monitoring.

You can curl the API directly to receive the information:

```
https://yourserver/admin/api/kcare/nagios/KCAREKEY
```

or you can use [https://patches.kernelcare.com/downloads/nagios/check_kcare](https://patches.kernelcare.com/downloads/nagios/check_kcare) script by modifying KEY_KCARE_NAGIOS_ENDPOINT in it to point to your server (change [https://cln.cloudlinux.com/clweb](https://cln.cloudlinux.com/clweb) (link to old UI) or [https://cln.cloudlinux.com/console/auth/login](https://cln.cloudlinux.com/console/auth/login) (link to new UI) with [https://yourserver/admin](https://yourserver/admin) ).

:::tip Note
Access using `PARTNER_LOGIN/TOKEN` is not supported by KernelCare.ePortal.
:::

## Usage reports

In common case, usage reports are sending automatically but when it's not
possible, ePortal will try to send report as an email. That requires configured
Sendmail (SSMTP) on the host. You can find a short instruction below.

If mail sending attempt fails, ePortal will save the reports in
`/usr/share/kcare-eportal/reports` which should be sent manually.

### How to configure Sendmail (SSMTP)

First of all you need to install `ssmtp`:

    yum install -y ssmtp

Edit `/etc/ssmtp/ssmtp.conf` file in accordance with your SMTP server
configuration. Here is a simple config file describing a common way to connect
to Gmail accounts:

    root=username@gmail.com
    mailhub=smtp.gmail.com:587
    hostname=localhost
    UseSTARTTLS=YES
    AuthUser=username@gmail.com
    AuthPass=xxxxxxxxxxxxxxxxxxx
    FromLineOverride=YES
    TLS_CA_File=/etc/ssl/certs/ca-certificates.crt

Actual location of a TLS_CA_Files depends on Linux distribution:

    "/etc/ssl/certs/ca-certificates.crt",                // Debian/Ubuntu/Gentoo etc.
    "/etc/pki/tls/certs/ca-bundle.crt",                  // Fedora/RHEL 6
    "/etc/ssl/ca-bundle.pem",                            // OpenSUSE
    "/etc/pki/tls/cacert.pem",                           // OpenELEC
    "/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem", // CentOS/RHEL 7

Also you could get if from:

    curl-config --ca

Now you can test a connection:

    echo -n 'Subject: test\n\nTesting ssmtp' | sendmail -v some-recipient@gmail.com

## Custom environment variables

You can define your own environment variables for ePortal process.

There is the `/usr/share/kcare-eportal/environment` folder, which is basically
daemon tools compatible envdir.

For example, to disable the default https verification you can set the
`PYTHONHTTPSVERIFY` environment variable to `0` as follows:

```
echo 0 > /usr/share/kcare-eportal/environment/PYTHONHTTPSVERIFY`
```


## Authentication using Single Sign On

ePortal supports Single Sign On authentication by "OAuth 2.0" standard, "Authorization Code" flow.

### Configuration

To enable SSO you need to configure OAuth2 authentication service provider and ePortal.

#### Okta Application setup example

You set up your OpenID Connect application inside the Okta Admin Console:

1. In the Admin Console, go to **Applications** > **Applications**.

2. Click **Create App Integration**.

3. On the Create a new app integration page, select **OIDC - OpenID Connect**
   for the **Sign-in method** and then select **Web Application** as the
   **Application type**.

4. Fill in the **Application Settings**.

| **Setting**           | **Value**
|-----------------------|-------------------------------------------
| Sign-in redirect URI  | http://eportal_ip/admin/sso/login/callback
| Sign-out redirect URI | http://eportal_ip/admin

5. Obtain *Client ID*, *Client secret*, the full URL to the `/authorize`, `/token`, `/userinfo` endpoints.

#### ePortal setup

On the ePortal machine, configure SSO settings.

Edit [ePortal config file](#config-files):

```
OIDC_AUTH_URL="https://dev-61441893.okta.com/oauth2/v1/authorize"
OIDC_TOKEN_URL="https://dev-61441893.okta.com/oauth2/v1/token"
OIDC_USERINFO_URL="https://dev-61441893.okta.com/oauth2/v1/userinfo"

OIDC_CLIENT_ID="0Aa134lzhUKj8jDMo5d7"
OIDC_CLIENT_SECRET="AoBNuWRLRu2dxIR3Q0btO53N1entmGxBjQqwmjVL"
```

Restart ePortal (see [Stopping & Starting](#stopping-starting)
section, choose a corresponding OS).

If all settings configured correctly the new **Sign In with SSO** button has to appear on login page `http://eportal_ip/admin/login`

![](/images/sso_eportal.png)


## Custom patches storage

By default patches with databases are stored in `/usr/share/kcare-eportal` for RHEL-based
systems and `/var/lib/eportal` for deb-based systems. You can configure it to
use split storage, for example to keep patches on external block device.

Edit [ePortal config file](#config-files):

```
PATCHES_DIR = '/path/to/patches/storage'
```

Run:

```
kc.eportal sync-nginx-conf
```

It initializes storage and makes corresponding modifications to nginx config to
serve patches from a new place.

And restart `nginx` and `eportal` services to apply changes:

```
systemctl restart eportal nginx
```
