---
layout: post
title: "Jaas multi-login (LDAP on TLS + AD + Realm) on CentOS 8"
date:   2022-05-07 00:00:00 -0000
categories: tutorial howto auth jaas
---

# Overview
Rundeck has the ability to integrate multiple authentication methods thanks to the Java Authentication and Authorization Service, or [JAAS](https://en.wikipedia.org/wiki/Java_Authentication_and_Authorization_Service) .
These authorization modules include ActiveDirectory, LDAP, and Realm which is Rundeck's default authentication.
There are other types of authentication like [Single-Sign-On](https://en.wikipedia.org/wiki/Single_sign-on) covered in another tutorial.

---

# What we'll learn
- Adding authentication for LDAP and ActiveDirectory servers, to coexist with local Realm module.
- Improve Realm authentication to make it hot-reloadable and avoid restarting Rundeck after every user/password/role change.
- Adding LDAP SSL certificate to Java's cacerts in system to make Rundeck correctly verify the server.
#- *multilogin.conf* used as custom file name which will override Jaas defaults.
#- *multilogin* used as custom module name which will override Jaas defaults.

---

# What we'll use
- An already deployed and configured Windows Domain to use it as ActiveDirectory server.
- An already deployed and configured LDAP server. The `osixia/openldap` docker image was used in this tutorial.
- CentOS `8` (or `7` will work as well) as the Rundeck server.
- Rundeck RPM install package (`3.4.10` was used).
- Java JRE or JDK version `1.8.0_302` or latest `8` branch. Also Java `11` will work as well.

---

# How it works by default
- Rundeck needs to restart if any changes are made in realm.properties file to make effect within the application.
- JAAS default file is `/etc/rundeck/jaas-loginmodule.conf` which uses `RDpropertyfilelogin` as module name with below contents:
```
RDpropertyfilelogin {
org.eclipse.jetty.jaas.spi.PropertyFileLoginModule required
debug="true"
file="/etc/rundeck/realm.properties";
};
```

The last line points to `/etc/rundeck/realm.properties` which contains the <user>,<password>,<role> definitions as below:
```
admin:admin,user,admin
user:user,user
```

---

# Steps
**1. Configure JAAS modules**
```
~$ cat << EOF > /etc/rundeck/multilogin.conf
multilogin {
	com.dtolabs.rundeck.jetty.jaas.JettyCachingLdapLoginModule sufficient
	debug="true"
	contextFactory="com.sun.jndi.ldap.LdapCtxFactory"
	providerUrl="ldaps://windowsdomain.local:636"
	bindDn="administrator@windowsdomain.local"
	bindPassword="mysecret"
	authenticationMethod="simple"
	forceBindingLogin="true"
	userBaseDn="CN=Users,DC=windowsdomain,DC=local"
	userRdnAttribute="sAMAccountName"
	userIdAttribute="sAMAccountName"
	userPasswordAttribute="unicodePwd"
	userObjectClass="user"
	supplementalRoles="user"
	roleBaseDn="CN=Administrators,CN=Builtin,DC=windowsdomain,DC=local"
	roleNameAttribute="cn"
	roleMemberAttribute="member"
	roleObjectClass="group"
	cacheDurationMillis="300000"
	reportStatistics="true";

        com.dtolabs.rundeck.jetty.jaas.JettyCombinedLdapLoginModule sufficient
        debug="true"
        contextFactory="com.sun.jndi.ldap.LdapCtxFactory"
        providerUrl="ldaps://ldap.local:636"
        bindDn="cn=admin,dc=ldap,dc=local"
	bindPassword="mysecret"
        authenticationMethod="simple"
        forceBindingLogin="false"
        userBaseDn="ou=Users,dc=ldap,dc=local"
        userRdnAttribute="uid"
        userIdAttribute="uid"
        userPasswordAttribute="userPassword"
        userObjectClass="posixAccount"
        roleBaseDn="cn=Roles,dc=ldap,dc=local"
        roleNameAttribute="cn"
        roleUsernameMemberAttribute="memberUid"
        roleMemberAttribute="memberUid"
        roleObjectClass="posixGroup"
        cacheDurationMillis="300000"
        supplementalRoles="user"
        reportStatistics="true"
        timeoutRead="10000"
        timeoutConnect="20000"
        nestedGroups="false";

        org.rundeck.jaas.jetty.ReloadablePropertyFileLoginModule optional
        debug="true"
        file="/etc/rundeck/realm.properties";
};
EOF
```

**2. Override default JAAS configuration with new ones in rundeckd file**
```
~$ cat << EOF > /etc/sysconfig/rundeckd
export JAAS_LOGIN=true
export JAAS_CONF=/etc/rundeck/multilogin.conf
export LOGIN_MODULE=multilogin
EOF
```

**3. Add LDAP SSL certificate to Java's cacerts keystore**
```
~$ keytool -printcert -sslserver ldap.local:636 > ldap.pem
~$ keytool -importcert -v -keystore /etc/pki/java/cacerts -file ldap.pem -storepass changeit -noprompt -dname CN=ldap.local
```

**3. Restart Rundeck and try authentications**
```
~$ service rundeckd restart && tail -f /var/log/rundeck/service.log
```

---

# Conclusion
After these steps you should be now able to authenticate to these multiple auth providers.

---

# Docs
[https://docs.rundeck.com/docs/](https://docs.rundeck.com/docs/)

![](/contents/rundeck.png)

