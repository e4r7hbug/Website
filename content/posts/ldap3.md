+++
title = "Using The Python ldap3 Package"
date = ""
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
+++

# LDAP Lookups For OpenVPN

My coworker was having some issues with the
[ldap3](https://ldap3.readthedocs.io/en/latest/index.html) Package behaving
oddly. This came about because OpenVPN Access Server 2.8.0 changed from
[python-ldap](https://www.python-ldap.org/) to `ldap3`.

The idea being, have OpenVPN authenticate with Okta and assign VPN Profiles
matching your Active Directory (AD) Group.

## The Problem

He was trying to query the Okta LDAP endpoint for a list of Groups that a User
belongs to.

## Solution

Once I installed [Apache Directory](https://directory.apache.org/studio/), I
could more easily view the query results. After some clicking around, I drilled
down to create a Search.

```
DIT
└── Root DSE (2)
  └── dc=business,dc=okta,dc=com
     └── ou=groups
         └── cn=vpn1
```

I right-clicked on `cn=vpn1` and selected `New -> New Search…`. This is where I
learned more about the `Search Base` and `Filter` fields.

* Search Base: `cn=vpn1,ou=groups,dc=business,dc=okta,dc=com`
* Filter: `(objectClass=*)`

Looking in the `Attribute Description` column of `cn=vpn1`, I noticed that
`objectClass` was one of the listed attributes. Then I thought, maybe this
thing can create the search for me if I can click on the line with peoples's
names? Right-clicking on the line with:

| Attribute Description | Value |
| --------------------- | ----- |
| uniqueMember | uid=me@business.com,ou=users,dc=business,dc=okta,dc=com' |

I was able to select `New Search…` and it had the fields populated for me.

* Search Base: `cn=vpn1,ou=groups,dc=business,dc=okta,dc=com`
* Filter: `(uniqueMember=uid=me@business.com,ou=users,dc=business,dc=okta,dc=com)`

This was close enough, but when I tested out the query with `ldap3`, I got an
object back that was essentially a string that we would need to parse.

```python
connection.search(
    search_base='cn=vpn1,ou=groups,dc=business,dc=okta,dc=com',
    search_filter='(uniqueMember=uid=me@business.com)',
)
# Search results are stored in `connection.entries` as a list

vpn_groups = set(entry.entry_dn for entry in connection.entries)

return vpn_groups
# {'cn=vpn1,ou=groups,dc=business,dc=okta,dc=com'}
```

That's when I remembered he was testing some stuff with attributes. In the
Search properties, I noticed there was a `Returning Attributes` field. I looked
that up and it's supposed to give you back the attributes you want from the
returned objects. So I tried adding `cn` as an attribute and got back a list
that didn't need parsing.

```python
connection.search(
    search_base='cn=vpn1,ou=groups,dc=business,dc=okta,dc=com',
    search_filter='(uniqueMember=uid=me@business.com)',
    attributes=['cn'],
)
# Search results are stored in `connection.entries` as a list

vpn_groups = set(entry.cn.value for entry in connection.entries)

return vpn_groups
# {'vpn1'}
```

Nice! But is there a way for me to get all of the Groups? Turns out, if you
remove the `cn=vpn1` from the Search Base, you query all of the Groups. With
this, we get a list of all Groups where I am a member.

```python
connection.search(
    search_base='ou=groups,dc=business,dc=okta,dc=com',
    search_filter='(uniqueMember=uid=me@business.com)',
    attributes=['cn'],
)
# Search results are stored in `connection.entries` as a list

all_groups = set(entry.cn.value for entry in connection.entries)

return all_groups
# {'all', 'california', 'devops', 'testvpn', 'productionvpn', 'vpn1'}
```

Even better! Now let's try and filter this back down to only Groups with the
word `vpn`. My coworker gave me the last piece of the Filter to glob match the
names. Doing `(&(cn=*vpn*)(uniqueMember=uid=me@business.com))` means we want
results where the Group name has `vpn` in it AND I am a member. So the final
piece looks about:

```python
"""Get Group names that a User belongs to using Okta LDAP.

MIT License

Copyright (c) 2020 Nate Tangsurat

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
"""
import ldap3

HOST = 'ldaps://business.ldap.okta.com'
USER = 'uid=secret_read_only@business.com,dc=business,dc=okta,dc=com'
PASSWORD = 'SUPA_SECRET_PARADISE_IN_VAULT'


def main():
    """Test LDAP query."""
    vpn_groups = set()

    connection = ldap3.Connection(HOST, user=USER, password=PASSWORD, auto_bind=True)

    search_base = 'ou=groups,dc=business,dc=okta,dc=com'
    search_filter = '(&(cn=*vpn*)(uniqueMember=uid={user}@business.com))'.format(user='me')

    query_ok = connection.search(
        search_base=search_base,
        search_filter=search_filter,
        attributes=['cn'],
    )
    # Search results are stored in `connection.entries` as a list

    if query_ok:
        vpn_groups = set(entry.cn.value for entry in connection.entries)
    else:
        print('Query did not return results.')

    return vpn_groups


if __name__ == '__main__':
    main()
```
