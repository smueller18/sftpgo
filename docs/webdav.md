# WebDAV

The experimental `WebDAV` support can be enabled setting a `bind_port` inside the `webdavd` configuration section.

Each user has his own path like `http/s://<SFTPGo ip>:<WevDAVPORT>/<username>` and it must authenticate using password credentials.

WebDAV is quite a different protocol than SCP/FTP, there is no session concept, each command is a separate HTTP request and must be authenticated, performance can be greatly improved enabling caching for the authenticated users (it is enabled by default). This way SFTPGo don't need to do a dataprovider query and a password check for each request.
If you enable quota support a dataprovider query is required, to update the user quota, after each file upload.

The caching configuration allows to set:

- `expiration_time` in minutes. If a user is cached for more than the specificied minutes it will be removed from the cache and a new dataprovider query will be performed. Please note that the `last_login` field will not be updated and `external_auth_hook`, `pre_login_hook` and `check_password_hook` will not be executed if the user is obtained from the cache.
- `max_size`. Maximum number of users to cache. When this limit is reached the user with the oldest expiration date will be removed from the cache. 0 means no limit however the cache size cannot exceed the number of users so if you have a small number of users you can leave this setting to 0.

Users are automatically removed from the cache after an update/delete.

WebDAV should work as expected for most use cases but there are some minor issues and some missing features.

Know issues:

- removing a directory tree on Cloud Storage backends could generate a `not found` error when removing the last (virtual) directory. This happen if the client cycles the directories tree itself and removes files and directories one by one instead of issuing a single remove command
- the used [WebDAV library](https://pkg.go.dev/golang.org/x/net/webdav?tab=doc) asks to open a file to execute a `stat` and sometime reads some bytes to find the content type. We are unable to distinguish a `stat` from a `download` for now, so to be able to proper list a directory you need to grant both `list` and `download` permissions
- the used `WebDAV library` not always returns a proper error code/message, most of the times it simply returns `Method not Allowed`. I'll try to improve the library error codes in the future
- if an object within a directory cannot be accessed, for example due to OS permissions issues or because is a missing mapped path for a virtual folder, the directory listing will fail. In SFTP/FTP the directory listing will succeed and you'll only get an error if you try to access to the problematic file/directory

We plan to add [Dead Properties](https://tools.ietf.org/html/rfc4918#section-3) support in future releases. We need a design decision here, probably the best solution is to store dead properties inside the data provider but this could increase a lot its size. Alternately we could store them on disk for local filesystem and add as metadata for Cloud Storage, this means that we need to do a separate `HEAD` request to retrieve dead properties for an S3 file. For big folders will do a lot of requests to the Cloud Provider, I don't like this solution. Another option is to expose a hook and allow you to implement `dead properties` outside SFTPGo.

If you find any other quircks or problems please let us know opening a GitHub issue, thank you!
