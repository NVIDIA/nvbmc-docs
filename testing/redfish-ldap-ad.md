# OpenBMC Redfish LDAP & Active Directory Support

**Document Purpose:** How to configure and use LDAP & AD features with OpenBMC Redfish Service

**Audience:** Developer/Tester familiar with Redfish Server, LDAP, AD

**Prerequisites:** LDAP & AD servers

**Author:** Tejas Patil

**Created:** 2021-12-07


# GET AccountService
   LDAP & AD related properties will be configured under Account Service.
   GET request on Account Service when there is no LDAP/AD configuration.
```
URI: /redfish/v1/AccountService/
METHOD: GET
```
   On successful user will receive the Account Service properties along with
   supported LDAP & AD properties.
```
{
  "@odata.id": "/redfish/v1/AccountService",
  "@odata.type": "#AccountService.v1_5_0.AccountService",
  "AccountLockoutDuration": 0,
  "AccountLockoutThreshold": 0,
  "Accounts": {
    "@odata.id": "/redfish/v1/AccountService/Accounts"
  },
  "ActiveDirectory": {
    "Authentication": {
      "AuthenticationType": "UsernameAndPassword",
      "Password": null,
      "Username": ""
    },
    "LDAPService": {
      "SearchSettings": {
        "BaseDistinguishedNames": [
          ""
        ],
        "GroupsAttribute": "",
        "UsernameAttribute": ""
      }
    },
    "RemoteRoleMapping": [],
    "ServiceAddresses": [
      ""
    ],
    "ServiceEnabled": false
  },
  "Description": "Account Service",
  "Id": "AccountService",
  "LDAP": {
    "Authentication": {
      "AuthenticationType": "UsernameAndPassword",
      "Password": null,
      "Username": ""
    },
    "Certificates": {
      "@odata.id": "/redfish/v1/AccountService/LDAP/Certificates"
    },
    "LDAPService": {
      "SearchSettings": {
        "BaseDistinguishedNames": [
          ""
        ],
        "GroupsAttribute": "",
        "UsernameAttribute": ""
      }
    },
    "RemoteRoleMapping": [],
    "ServiceAddresses": [
      ""
    ],
    "ServiceEnabled": false
  },
  "MaxPasswordLength": 20,
  "MinPasswordLength": 8,
  "Name": "Account Service",
  "Oem": {
    "OpenBMC": {
      "@odata.id": "/redfish/v1/AccountService#/Oem/OpenBMC",
      "@odata.type": "#OemAccountService.v1_0_0.AccountService",
      "AuthMethods": {
        "BasicAuth": true,
        "Cookie": true,
        "SessionToken": true,
        "TLS": true,
        "XToken": true
      }
    }
  },
  "Roles": {
    "@odata.id": "/redfish/v1/AccountService/Roles"
  },
  "ServiceEnabled": true
}
```

## LDAP Configuration
1. Configure LDAP Server by PATCH on AccountService.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "LDAP": {
        "ServiceAddresses": ["ldap://172.31.2.79/"],
        "Authentication": {
            "AuthenticationType":"UsernameAndPassword",
            "Username": "cn=admin,dc=amimegarac,dc=com",
            "Password": "admin"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": ["dc=amimegarac,dc=com"]           
            }
        }    
    }
}
```
     On successful AccountService will return 200(OK) with the json response
     of LDAP properties.
```
{
    "LDAP": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": "",
            "Username": "cn=admin,dc=amimegarac,dc=com"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=amimegarac,dc=com"
                ],
                "GroupsAttribute": "",
                "UsernameAttribute": ""
            }
        },
        "RemoteRoleMapping": [],
        "ServiceAddresses": [
            "ldap://172.31.2.79/"
        ],
        "ServiceEnabled": false
    }
}
```

2. Add the Remote Role Map and enable the LDAP service by PATCH on AccountService.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
  "LDAP": {    
    "RemoteRoleMapping": [
      {
        "LocalRole": "Administrator",
        "RemoteGroup": "web"
      }
    ],
    "ServiceEnabled": true
  }
}
```
     On successful AccountService will return 200(OK) with the json response
     of LDAP properties.
```
{
    "LDAP": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": null,
            "Username": "cn=admin,dc=amimegarac,dc=com"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=amimegarac,dc=com"
                ],
                "GroupsAttribute": "",
                "UsernameAttribute": ""
            }
        },
        "RemoteRoleMapping": [
            {
                "LocalRole": "Administrator",
                "RemoteGroup": "web"
            }
        ],
        "ServiceAddresses": [
            "ldap://172.31.2.79/"
        ],
        "ServiceEnabled": true
    }
}
```

3. GET AccountService after patching LDAP configuration successfully.
```
URI: /redfish/v1/AccountService/
METHOD: GET
```
   On successful user will receive the ldap configuration detials.
```
{
    "@odata.id": "/redfish/v1/AccountService",
    "@odata.type": "#AccountService.v1_5_0.AccountService",
    "AccountLockoutDuration": 0,
    "AccountLockoutThreshold": 0,
    "Accounts": {
        "@odata.id": "/redfish/v1/AccountService/Accounts"
    },
    "ActiveDirectory": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": null,
            "Username": ""
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    ""
                ],
                "GroupsAttribute": "",
                "UsernameAttribute": ""
            }
        },
        "RemoteRoleMapping": [],
        "ServiceAddresses": [
            ""
        ],
        "ServiceEnabled": false
    },
    "Description": "Account Service",
    "Id": "AccountService",
    "LDAP": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": null,
            "Username": "cn=admin,dc=amimegarac,dc=com"
        },
        "Certificates": {
            "@odata.id": "/redfish/v1/AccountService/LDAP/Certificates"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=amimegarac,dc=com"
                ],
                "GroupsAttribute": "gidNumber",
                "UsernameAttribute": "cn"
            }
        },
        "RemoteRoleMapping": [
            {
                "LocalRole": "Administrator",
                "RemoteGroup": "web"
            }
        ],
        "ServiceAddresses": [
            "ldap://172.31.2.79/"
        ],
        "ServiceEnabled": true
    },
    "MaxPasswordLength": 20,
    "MinPasswordLength": 8,
    "Name": "Account Service",
    "Oem": {
        "OpenBMC": {
            "@odata.id": "/redfish/v1/AccountService#/Oem/OpenBMC",
            "@odata.type": "#OemAccountService.v1_0_0.AccountService",
            "AuthMethods": {
                "BasicAuth": true,
                "Cookie": true,
                "SessionToken": true,
                "TLS": true,
                "XToken": true
            }
        }
    },
    "Roles": {
        "@odata.id": "/redfish/v1/AccountService/Roles"
    },
    "ServiceEnabled": true
}
```

4. Login with LDAP user credentials.
```
URI: /redfish/v1/AccountService/
METHOD: GET
UserName: ldap username (ami)
Password: ldap passoword (ami123)
```
   On successful user will receive the Account Service response.
   LDAP user must be able to access all Redfish URIs along with WebUI session.


# LDAP Negative Test Cases
1. Configure LDAP server with empty json.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "LDAP": {
        "Authentication": {}
        }
}
```
     This request will fail with 400(Bad Request) return code.

2. Configure LDAP server with empty server address.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "LDAP": {
        "ServiceAddresses": [null]
    }
}
```
     This request will fail with 400(Bad Request) return code.

3. Configure LDAP server with invalid AuthenticationType.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "LDAP": {
        "Authentication": {
            "AuthenticationType": "abcdef"
        }
    }
}
```
     This request will fail with 400(Bad Request) return code.


# Active Directory Configuration
1. Configure AD Server by PATCH on AccountService.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "ActiveDirectory": {
        "ServiceAddresses": ["ldap://172.31.1.156/"],
        "Authentication": {
            "AuthenticationType":"UsernameAndPassword",
            "Username": "cn=Administrator,cn=Users,dc=osp,dc=mrac",
            "Password": "Megarac123!"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": ["dc=osp,dc=mrac"]
            }
        }
    }
}
```
     On successful AccountService will return 200(OK) with the json response
     of AD properties.
```
{
    "ActiveDirectory": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": "",
            "Username": "cn=Administrator,cn=Users,dc=osp,dc=mrac"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=osp,dc=mrac"
                ],
                "GroupsAttribute": "",
                "UsernameAttribute": ""
            }
        },
        "RemoteRoleMapping": [],
        "ServiceAddresses": [
            "ldap://172.31.1.156/"
        ],
        "ServiceEnabled": false
    }
}
```

2. Add the Remote Role Map and enable the AD service by PATCH on AccountService.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "ActiveDirectory": {
        "RemoteRoleMapping": [
            {
                "LocalRole": "Administrator",
                "RemoteGroup": "osp"
            }
        ],
        "ServiceEnabled": true
    }
}
```
     On successful AccountService will return 200(OK) with the json response
     of AD properties.
```
{
    "ActiveDirectory": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": null,
            "Username": "cn=Administrator,cn=Users,dc=osp,dc=mrac"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=osp,dc=mrac"
                ],
                "GroupsAttribute": "primaryGroupID",
                "UsernameAttribute": "sAMAccountName"
            }
        },
        "RemoteRoleMapping": [
            {
                "LocalRole": "Administrator",
                "RemoteGroup": "osp"
            }
        ],
        "ServiceAddresses": [
            "ldap://172.31.1.156/"
        ],
        "ServiceEnabled": true
    }
}
```

3. GET AccountService after patching AD configuration successfully.
```
URI: /redfish/v1/AccountService/
METHOD: GET
```
   On successful user will receive the AD configuration detials.
```
{
    "@odata.id": "/redfish/v1/AccountService",
    "@odata.type": "#AccountService.v1_5_0.AccountService",
    "AccountLockoutDuration": 0,
    "AccountLockoutThreshold": 0,
    "Accounts": {
        "@odata.id": "/redfish/v1/AccountService/Accounts"
    },
    "ActiveDirectory": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": null,
            "Username": "cn=Administrator,cn=Users,dc=osp,dc=mrac"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=osp,dc=mrac"
                ],
                "GroupsAttribute": "primaryGroupID",
                "UsernameAttribute": "sAMAccountName"
            }
        },
        "RemoteRoleMapping": [
            {
                "LocalRole": "Administrator",
                "RemoteGroup": "osp"
            }
        ],
        "ServiceAddresses": [
            "ldap://172.31.1.156/"
        ],
        "ServiceEnabled": true
    },
    "Description": "Account Service",
    "Id": "AccountService",
    "LDAP": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": null,
            "Username": ""
        },
        "Certificates": {
            "@odata.id": "/redfish/v1/AccountService/LDAP/Certificates"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    ""
                ],
                "GroupsAttribute": "",
                "UsernameAttribute": ""
            }
        },
        "RemoteRoleMapping": [],
        "ServiceAddresses": [
            ""
        ],
        "ServiceEnabled": false
    },
    "MaxPasswordLength": 20,
    "MinPasswordLength": 8,
    "Name": "Account Service",
    "Oem": {
        "OpenBMC": {
            "@odata.id": "/redfish/v1/AccountService#/Oem/OpenBMC",
            "@odata.type": "#OemAccountService.v1_0_0.AccountService",
            "AuthMethods": {
                "BasicAuth": true,
                "Cookie": true,
                "SessionToken": true,
                "TLS": true,
                "XToken": true
            }
        }
    },
    "Roles": {
        "@odata.id": "/redfish/v1/AccountService/Roles"
    },
    "ServiceEnabled": true
}
```

4. Add the same AD Role Group in BMC with Group Id of 513.
   `groupadd -g 513 <role_group>`
```
Example: groupadd -g 513 osp
```

5. Login with AD user credentials.
```
URI: /redfish/v1/AccountService/
METHOD: GET
UserName: ldap username (ospUser1)
Password: ldap passoword (Megarac123!)
```
   On successful user will receive the Account Service response.
   AD user must be able to access all Redfish URIs along with WebUI session.


# Active Directory different Test scenarios
1. Remove Role Mapping.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "ActiveDirectory": {
        "RemoteRoleMapping": [null]
    }
}
```
     On successful user will receive 200(OK) return code with AD properties
     response and the RemoteRoleMapping will be deleted.
```
{
    "ActiveDirectory": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": null,
            "Username": "cn=Administrator,cn=Users,dc=osp,dc=mrac"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=osp,dc=mrac"
                ],
                "GroupsAttribute": "primaryGroupID",
                "UsernameAttribute": "sAMAccountName"
            }
        },
        "RemoteRoleMapping": [
            null
        ],
        "ServiceAddresses": [
            "ldap://172.31.1.156/"
        ],
        "ServiceEnabled": true
    }
}
```

2. Enable LDAP, if AD is already enabled.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "LDAP": {
        "ServiceEnabled":true,
        "ServiceAddresses": ["ldap://172.31.2.79/"],
        "RemoteRoleMapping": [
            {
                "LocalRole": "Administrator",
                "RemoteGroup": "web"
            }
        ],
        "Authentication": {
            "AuthenticationType":"UsernameAndPassword",
            "Username": "cn=admin,dc=amimegarac,dc=com",
            "Password": "admin"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": ["dc=amimegarac,dc=com"]           
            }
        }    
    }
}
```
     This request will fail with 500(Internal Server Error) return code with error response.
     If AD is already enabled then we can't enable LDAP at that time and vice versa.
     To enable LDAP, we must first disbale AD first.
```
{
    "LDAP": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": "",
            "Username": "cn=admin,dc=amimegarac,dc=com"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=amimegarac,dc=com"
                ],
                "GroupsAttribute": "",
                "UsernameAttribute": ""
            }
        },
        "RemoteRoleMapping": [
            {
                "LocalRole": "Administrator",
                "RemoteGroup": "web"
            }
        ],
        "ServiceAddresses": [
            "ldap://172.31.2.79/"
        ],
        "ServiceEnabled": false
    },
    "error": {
        "@Message.ExtendedInfo": [
            {
                "@odata.type": "#Message.v1_1_1.Message",
                "Message": "The request failed due to an internal service error.  The service is still operational.",
                "MessageArgs": [],
                "MessageId": "Base.1.8.1.InternalError",
                "MessageSeverity": "Critical",
                "Resolution": "Resubmit the request.  If the problem persists, consider resetting the service."
            }
        ],
        "code": "Base.1.8.1.InternalError",
        "message": "The request failed due to an internal service error.  The service is still operational."
    }
}
```

3. Disable AD first then Enable LDAP.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "ActiveDirectory": {
        "ServiceEnabled": false
    }
}
```
     This request will disable the AD server then user can configure and enable LDAP server.
```
URI: /redfish/v1/AccountService/
METHOD: PATCH
REQUEST BODY:
{
    "LDAP": {
        "ServiceEnabled":true,
        "ServiceAddresses": ["ldap://172.31.2.79/"],
        "RemoteRoleMapping": [
            {
                "LocalRole": "Administrator",
                "RemoteGroup": "web"
            }
        ],
        "Authentication": {
            "AuthenticationType":"UsernameAndPassword",
            "Username": "cn=admin,dc=amimegarac,dc=com",
            "Password": "admin"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": ["dc=amimegarac,dc=com"]           
            }
        }    
    }
}
```
     On successful AccountService will return 200(OK) with the json response
     of LDAP properties and LDAP server will be enabled.
```
{
    "ActiveDirectory": {
        "Authentication": {
            "AuthenticationType": "UsernameAndPassword",
            "Password": "",
            "Username": "cn=Administrator,cn=Users,dc=osp,dc=mrac"
        },
        "LDAPService": {
            "SearchSettings": {
                "BaseDistinguishedNames": [
                    "dc=osp,dc=mrac"
                ],
                "GroupsAttribute": "",
                "UsernameAttribute": ""
            }
        },
        "RemoteRoleMapping": [],
        "ServiceAddresses": [
            "ldap://172.31.1.156/"
        ],
        "ServiceEnabled": false
    }
}
```

## Configure OpenLDAP in Linux

1. apt-get update

2. apt-get install slapd ldap-utils

3. dpkg-reconfigure slapd
   Configure the slapd with proper DNS domain name, organization name and password.

4. apt-get install phpldapadmin

5. Replace below data inside "/etc/phpldapadmin/config.php" file.
   (conside here DN=megaracoe and ldap server ip=172.31.2.79)

   i) Search for "$servers->setValue('server','host','127.0.0.1');"
      and replace "127.0.0.1" with LDAP server IP.
      After changes it should look like this.
```
      "$servers->setValue('server','host','172.31.2.79');"
```
   ii) Search for "$servers>setValue('server','base',array('dc=example,dc=com'));"
       and "$servers->setValue('login','bind_id','cn=admin,dc=example,dc=com');"
       and replace "dc=example" with DNS domain name.
       After changes it should look like this.
```
       "$servers>setValue('server','base',array('dc=megaracoe,dc=com'));"
       "$servers->setValue('login','bind_id','cn=admin,dc=megaracoe,dc=com');"
```
   iii) Search for the "$config->custom->appearance['hide_template_warning'] = false;"
        and uncomment it and set it to true, to avoid some annoying warnings that are unimportant.
   iv) Save and close the file.

6. Login to web interface of LDAP Client.
   In browser, Hit the URL "<IP>/phpldapadmin" or "<domain_name>/phpldapadmin"
   Login with DN and password.

7. Create LDIF data for uses for root domain.
   i) Create file with ".ldif" (eg: add_entry.ldif) extension.
   ii) Add the below data in that file.
```
       dn: cn=testUser,cn=admin,dc=megaracoe,dc=com
       objectClass: uidObject
       objectClass: top
       objectClass: person
       objectClass: posixAccount
       cn: testUser
       sn: testUser
       uid: testUser
       userPassword: Nvidia_123
       uidNumber: 10
       gidNumber: 1003
       homeDirectory: /home/testUser
```
  iii) Add the ldap user.
```
       ldapadd -x -D cn=admin,dc=megaracoe,dc=com -W -f add_entry.ldif
```
       With this user "testUser" will be added in the domain "megaracoe.com".


## Configure Active Directory in Windows

1. Open the server manager and goto PowerShell (as Administrator) and launch "ServerManager.exe".

2. On server manager, add roles & features. In the installation process select below options.
   i) Select Role based or feature based installation.
   ii) Choose select server from server pool.
   iii) Select the "Active Directory Domain Services" from the Server Roles, and add the features.

3. After installtion, open "Promote this server to a domain controller".

4. Select "Add a new forest" option under "Deployment Configuration".
   And enter "Root domain name" and password (eg: Root domain name: rebeladmin.net).

5. After the installation system will restart automatically. Once it booted, log in to the server as domain admin.

6. Add group.
   i) Open Server Manager-> Tools -> Active Directory Users and Computers.
   ii) Click Domain name and right click Users->New->Group
       (eg: Group Name=WebAdmin, Group Scope=Global, Group Type=Security)
       WebAdmin group has been added after this.

7. Add user.
   i) Right click on domain name and add user from New.
      Add all the details as First Name, Last Name & password.
      After this, New user has been added.

8. Add user to group.
   i) Right click on user and choose "Add to group".
   ii) Enter the group name.
       After this, user will be added to group successfully.
