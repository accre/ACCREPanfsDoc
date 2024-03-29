# ACCREPanfsDoc

## Extended attributes

Extended attributes of PanFS™ objects may be read or set with the standard Linux getfattr and setfattr commands. 
The current list of supported extended attributes is as follows:

    Purpose
    user.panfs.acl
        
    ACL controlling access permissions to the object.
    user.panfs.agg_layout_info
        
    RAID layout of the object on storage (See KB 294845 for details).
    user.panfs.flags
        
    Attributes of a file stored with flags, such as the archive, hidden, and system attributes.
    user.panfs.mgr_id
        
    File Manager (FM) ID associated with the Panasas director node volume service manager (VSM) hosting the volume containing the object.
    user.panfs.obj_id
        
    The internal PanFS object ID for the object.
    user.panfs.time_create
        
    The time the object was created, in Unix epoch format.
  
Below is an example of reading the user.panfs.acl extended attribute on the file file1:

    [root@service-quad2-b ~]#  getfattr -n user.panfs.acl /panfs/service-7.palab.panasas.com/test/file1
    getfattr: Removing leading '/' from absolute path names
    # file: panfs/service-7.palab.panasas.com/test/file1
    user.panfs.acl="v2: +uid:1795,rwakponNRWP* +gid:100,rnRP* +pan_gid:1,rnRP +uid:0,rwacdCnNDRWP,I:ID +gid:0,rwacdCnNDRWP,I:ID"

Below is an example of setting the user.panfs.acl extended attribute on the file file1, to revoke all permissions for the user with UID 1795:

    [root@service-quad2-b ~]# setfattr -n user.panfs.acl -v "v2: +gid:100,rnRP* +pan_gid:1,rnRP +uid:0,rwacdCnNDRWP,I:ID* +gid:0,rwacdCnNDRWP,I:ID" /panfs/service-7.palab.panasas.com/test/file1
    [root@service-quad2-b ~]#  getfattr -n user.panfs.acl /panfs/service-7.palab.panasas.com/test/file1
    getfattr: Removing leading '/' from absolute path names
    # file: panfs/service-7.palab.panasas.com/test/file1
    user.panfs.acl="v2: +gid:100,rnRP* +pan_gid:1,rnRP +uid:0,rwacdCnNDRWP,I:ID* +gid:0,rwacdCnNDRWP,I:ID"

## Background
PanFS® supports ACLs on both directories and file objects.  Every object has an Access Control List (ACL) that contains an ordered list of Access Control Entries (ACEs).  Each ACE specifies a type of permission, optional inheritance flags, an identity to grant permissions to, and a bitmask of permissions to grant. 

### Types of ACEs

    Positive (+):  Rights that are specifically given to an entity.
    Negative(~):  Rights that are specifically denied to an entity. 
    Audit(@):  Provided for Windows compatibility and intentionally not interpreted or given any specific meaning by PanFS.
    Alarm(!):  Also provided for Windows compatibility and intentionally not interpreted or given any specific meaning by PanFS.

### PanFS ACL Inheritance
The ACL inheritance feature is based on Windows and NFS v3 semantics.  When objects are created in any directory, the ACEs that have inheritance flags set are applied to the object being created. The following are the various inheritance flags defined. Each one has different semantics based on the object type that is being created.

Inheritance flags:

    OI (Object Inherit):  Objects (files in the directory) inherit the ACE
    CI (Container Inherit):  Subdirectories inherit the ACE ACE
    IO (Inherit Only):  Objects inherit the ACE but the ACE does not apply to this object
    NP (No Propagate):  The ACE will be inherited only one level down
    ID (Inherited):  The ACE was inherited from its parent

Each of these ACEs shows its type, identity, permission, and inheritance flag.

    4  +group_sid:S-1-5-21-1214440339,:all:,I:OICI
    1  +uid:0,:all:,I:OICI 
    2  +gid:0,rxnRP,I:OICI 
    3  +pan_gid:1,rxnRP,I:OICI 

ACE 4:  Windows users that belong to group S-1-5-21-1214440339 have all permissions.  This ACE will be inherited by this object and its subdirectories. 

ACE 1   Unix root user has all permissions.  This ACE will be inherited by this object and its subdirectories. 

ACE 2:  Unix users that belong to group 0 can read, execute, create, read attributes, and read the ACL.  The ACE will be inherited by this object and its subdirectories. 

ACE 3:  The everyone group (pan_gid:1) can read, execute, create, read attributes, and read the ACL.  The ACE will be inherited by this object and its subdirectories. 

### PanFS Identity Types

| Identity Type | Description              |
| ------------- | ------------------------ |
| uid:XXX       | Unix / NIS UID           |
| gid:XXX       | Unix / NIS GID           |
| user_sid:XXX  | Windows user SID         |
| group_sid:XXX | Windows group SID        |
| pan_gid:XXX   | Internal PanFS group     |

 
### PanFS Permissions 

| Permission | Abbreviation | Effect on File | Effect on Directory |
|------------|--------------|----------------|---------------------|
| read / list dir | r | read data contents | list dir |
| write | w | modify data contents | N/A |
| execute | x | read data contents | list dir |
| append | a | append data to object | N/A |
| create | c | N/A | create files |
| delete | d | delete files | delete dir |
| lock | k | take advisory or exclusive locks | N/A |
| change ACL | p | modify ACL | modify ACL |
| take ownership | o | change owner | change owner |
| sandbox | s | N/A | See Note 1 below. |
| create subdirectories | C | N/A | create subdirectories |
| read named attributes | n | read named attributes | N/A |
| write named attributes | N | write named attributes | N/A |
| delete child | D | N/A | delete files or subdirectories |
| read attributes | R | read attributes | read attributes |
| write attributes | W | write attributes | write attributes |
| read ACL | P | read ACL | read ACL |
| synchronize | S | N/A | N/A |
| write retention | h | N/A | N/A |
| write retention hold | H | N/A | N/A |

### Example Usage:

To view the original acl of a file or folder please use this command:
```
getfattr --dump /path/to/file/or/folder
```

For example:

```
$ ls -l
-rw-r--r-- 1 koirap1 accre   5 Feb 27 10:12 readthis

$ getfattr --dump readthis
Hide quoted text
# file: readthis
user.panfs.acl="v2: +uid:906449,rwakponNRWP* +gid:36014,rnRP* +pan_gid:1,rnRP"
...
```

Notice the `*(stars)`. They represent the primary ownership. Since it's the user `koirap1`, that uid and gid has primary ownership. The group can only r (read), n (read named attributes), R (read attributes) and P (read acls). For further explanation of the flags, refer to the document's previous sections.

Now to set the acl to allow (+uid) or disallow (~uid) a certain user from accessing the folder. Let's assume I want to allow user `vunetid` to access that file.

```
$ id vunetid
uid=315497(vunetid) ...
```

I would simply add the following acl to the file: `+uid:315497,rwakponNRWP`. Please notice that I want to add it to the already existing acls but the setfattr command replaces the entire acl. Therefore I would do this command.
```
$ setfattr -n user.panfs.acl -v "v2: +uid:906449,rwakponNRWP* +gid:36014,rnRP* +pan_gid:1,rnRP +uid:315497,rwakponNRWP" readthis
```

Now if you getfattr --dump on the file, you will see the following acls:
```
# file: readthis
user.panfs.acl="v2: +uid:315497,rwakponNRWP +uid:906449,rwakponNRWP* +gid:36014,rnRP* +pan_gid:1,rnRP"
```
suggesting that user with uid 315497 (chaudhu) no can perform the set flags.

Finally, one caveat here, is if you want to allow users to recursive access to folders. In that case, you would first make the folder traversable by setting the x flag to user, group or anyone outside. Only after that, can you do something like this:
```
$ mkdir test
$ ls -l  
total 56
-rw-r--r-- 1 koirap1 accre    5 Feb 27 10:12 readthis
drwxr-xr-x 2 koirap1 accre 4096 Feb 27 10:36 test

$ getfattr --dump test
user.panfs.acl="v2: +uid:906449,rxcdpoCnNDRWP* +gid:36014,rxnRP* +pan_gid:1,rxnRP +uid:0,rwxacdCnNDRWP,I:OICIID +gid:0,rxnRP,I:OICIID +pan_gid:1,rxnRP,I:OICIIOID"
```

The gid:0 is for root users and pan_gid:1 is for all users under the system. We can leave them as it is.
The only thing we need to change in here is to make sure that the user 315497 (vunetid) can read write and access the files inside the test folder. So we will add a `+uid:315497,rxcdpoCnNDRWP,I:OICI`. This translates to give uid 315497 access `rxcdpoCnNDRWP` and make this acl to be inherited by `objects (OI)` and `directories (CI)`. Refer to the previous section of this document for more details

```
$ setfattr -n user.panfs.acl -v "v2: +uid:906449,rxcdpoCnNDRWP* +gid:36014,rxnRP* +pan_gid:1,rxnRP +uid:0,rwxacdCnNDRWP,I:OICIID +gid:0,rxnRP,I:OICIID +pan_gid:1,rxnRP,I:OICIIOID +uid:315497,rxcdpoCnNDRWP,I:OICI"  test
```

Now you can create a file inside the folder and look at it's acls.
```
$ touch test/test.test
$ getfattr --dump test/test.test
# file: test.test
user.panfs.acl="v2: +uid:906449,rwakponNRWP* +gid:36014,rnRP* +pan_gid:1,rnRP +uid:315497,rcdpoCnNDRWP,I:ID +uid:0,rwacdCnNDRWP,I:ID +gid:0,rnRP,I:ID"
```
The `+uid:315497,rcdpoCnNDRWP,I:ID` part means that user 315497 has access `rcdpoCnNDRWP` and this acl is inherited from the parent folder's acl (I: ID)

