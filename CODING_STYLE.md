#### General Notes

- The coding style guide refers to new code. But legacy code can be cleaned up
  and we are happy to take those patches.
- Just because there is still code in LXC that doesn't adhere to the coding
  standards outlined here does not license not adhering to the coding style. In
  other words: please stick to the coding style.
- Maintainers are free to ignore rules specified here when merging pull
  requests. This guideline might seem a little weird but it exits to ease new
  developers into the code base and to prevent unnecessary bikeshedding. If
  a maintainer feels hat enforcing a specific rule in a given commit would do
  more harm than good they should always feel free to ignore the rule.

  Furthermore, when merging pull requests that do not adhere to our coding
  style maintainers should feel free to grab the commit, adapt it to our coding
  style and add their Signed-off-by line to it. This is especially helpful to
  make it easier for first-time contributors and to prevent having pull
  requests being stuck in the merge queue because of minor details.

#### Only Use Tabs

- LXC uses tabs.

#### Only use `/* */` Style Comments

- Any comments that are added must use `/* */`.
- All comments should start on the same line as the opening `/*`.
- Single-line comments should simply be placed between `/* */`. For example:
  ```
  /* Define pivot_root() if missing from the C library */
  ```
- Multi-line comments should end with the closing `*/` on a separate line. For
  example:
  ```
  /* At this point the old-root is mounted on top of our new-root
   * To unmounted it we must not be chdir()ed into it, so escape back
   * to old-root.
   */
  ```

#### Try To Wrap At 80chars

- This is not strictly enforced. It is perfectly valid to sometimes
  overflow this limit if it helps clarity. Nonetheless, try to stick to it
  and use common sense to decide when not to.

#### Error Messages

- Error messages must start with a capital letter and must **not** end with a
  punctuation sign.
- They should be descriptive, without being needlessly long. It is best to just
  use already existing error messages as examples.
- Examples of acceptable error messages are:
  ```
  SYSERROR("Failed to create directory \"%s\"", path);
  WARN("\"/dev\" directory does not exist. Proceeding without autodev being set up");
  ```

#### Return Error Codes

- When writing a function that can fail in a non-binary way try to return
  meaningful negative error codes (e.g. `return -EINVAL;`).

#### All Unexported Functions Must Be Declared `static`

- Functions which are only used in the current file and are not exported
  within the codebase need to be declared with the `static` attribute.

#### All Exported Functions Must Be Declared `extern` In A Header File

- Functions which are used in different files in the library should be declared
  in a suitable `*.c` file and exposed in a suitable `*.h` file. When defining
  the function in the `*.c` file the function signature should not be preceded
  by the `extern` keyword. When declaring the function signature in the `*.h`
  file it must be preceded by the `extern` keyword. For example:
  ```
  /* Valid function definition in a *.c file */
  ssize_t lxc_write_nointr(int fd, const void* buf, size_t count)
  {
          ssize_t ret;
  again:
          ret = write(fd, buf, count);
          if (ret < 0 && errno == EINTR)
                  goto again;
          return ret;
  }

  /* Valid function declaration in a *.h file */
  extern ssize_t lxc_write_nointr(int fd, const void* buf, size_t count);
  ```

#### All Names Must Be In lower_case

- All functions and variable names must use lower case.

#### Declaring Variables

- variables should be declared at the top of the function or at the beginning
  of a new scope but **never** in the middle of a scope
1. uninitialized variables
  - put base types before complex types
  - put standard types defined by libc before types defined by LXC
  - put multiple declarations of the same type on the same line
2. initialized variables
  - put base types before complex types
  - put standard types defined by libc before types defined by LXC
  - put multiple declarations of the same type on the same line
- Examples of good declarations can be seen in the following function:
  ```
  int lxc_clear_procs(struct lxc_conf *c, const char *key)
  {
          struct lxc_list *it, *next;
          bool all = false;
          const char *k = NULL;

          if (strcmp(key, "lxc.proc") == 0)
                  all = true;
          else if (strncmp(key, "lxc.proc.", sizeof("lxc.proc.") - 1) == 0)
                  k = key + sizeof("lxc.proc.") - 1;
          else
                  return -1;

          lxc_list_for_each_safe(it, &c->procs, next) {
                  struct lxc_proc *proc = it->elem;

                  if (!all && strcmp(proc->filename, k) != 0)
                          continue;
                  lxc_list_del(it);
                  free(proc->filename);
                  free(proc->value);
                  free(proc);
                  free(it);
          }

          return 0;
  }
    ```

#### Single-line `if` blocks should not be enclosed in `{}`

- This also affects `if-else` ladders if and only if all constituting
  conditions are
  single-line conditions. If there is at least one non-single-line
  condition `{}` must be used.
- For example:
  ```
  /* no brackets needed */
  if (size > INT_MAX)
          return -EFBIG;

  /* The else branch has more than one-line and so needs {}. This entails that
   * the if branch also needs to have {}.
   */
  if ( errno == EROFS ) {
          WARN("Warning: Read Only file system while creating %s", path);
  } else {
          SYSERROR("Error creating %s", path);
          return -1;
  }

  /* also fine */
  for (i = 0; list[i]; i++)
        if (strcmp(list[i], entry) == 0)
                return true;

  /* also fine */
  if (ret < 0)
          WARN("Failed to set FD_CLOEXEC flag on slave fd %d of "
               "pty device \"%s\": %s", pty_info->slave,
               pty_info->name, strerror(errno));

  /* also fine */
  if (ret == 0)
          for (i = 0; i < sizeof(limit_opt)/sizeof(limit_opt[0]); ++i) {
                  if (strcmp(res, limit_opt[i].name) == 0)
                          return limit_opt[i].value;
          }
  ```

#### Functions Not Returning Booleans Must Assign Return Value Before Performing Checks

- When checking whether a function not returning booleans was successful or not
  the returned value must be assigned before it is checked (`str{n}cmp()`
  functions being one notable exception). For example:
  ```
  /* assign value to "ret" first */
  ret = mount(sourcepath, cgpath, "cgroup", remount_flags, NULL);
  /* check whether function was successful */
  if (ret < 0) {
          SYSERROR("Failed to remount \"%s\" ro", cgpath);
          free(sourcepath);
          return -1;
  }
  ```
  Functions returning booleans can be checked directly. For example:
  ```
  extern bool lxc_string_in_array(const char *needle, const char **haystack);

  /* check right away */
  if (lxc_string_in_array("ns", (const char **)h->subsystems))
          continue;
  ```

#### Non-Boolean Functions That Behave Like Boolean Functions Must Explicitly Check Against A Value

- This rule mainly exists for `str{n}cmp()` type functions. In most cases they
  are used like a boolean function to check whether a string matches or not.
  But they return an integer. It is perfectly fine to check `str{n}cmp()`
  functions directly but you must compare explicitly against a value. That is
  to say, while they are conceptually boolean functions they shouldn't be
  treated as such since they don't really behave like boolean functions. So
  `if (!str{n}cmp())` and `if (str{n}cmp())` checks must not be used. Good
  examples are found in the following functions:
  ```
  static int set_config_hooks(const char *key, const char *value,
                              struct lxc_conf *lxc_conf, void *data)

          char *copy;

          if (lxc_config_value_empty(value))
                  return lxc_clear_hooks(lxc_conf, key);

          if (strcmp(key + 4, "hook") == 0) {
                  ERROR("lxc.hook must not have a value");
                  return -1;
          }

          copy = strdup(value);
          if (!copy)
                  return -1;

          if (strcmp(key + 9, "pre-start") == 0)
                  return add_hook(lxc_conf, LXCHOOK_PRESTART, copy);
          else if (strcmp(key + 9, "start-host") == 0)
                  return add_hook(lxc_conf, LXCHOOK_START_HOST, copy);
          else if (strcmp(key + 9, "pre-mount") == 0)
                  return add_hook(lxc_conf, LXCHOOK_PREMOUNT, copy);
          else if (strcmp(key + 9, "autodev") == 0)
                  return add_hook(lxc_conf, LXCHOOK_AUTODEV, copy);
          else if (strcmp(key + 9, "mount") == 0)
                  return add_hook(lxc_conf, LXCHOOK_MOUNT, copy);
          else if (strcmp(key + 9, "start") == 0)
                  return add_hook(lxc_conf, LXCHOOK_START, copy);
          else if (strcmp(key + 9, "stop") == 0)
                  return add_hook(lxc_conf, LXCHOOK_STOP, copy);
          else if (strcmp(key + 9, "post-stop") == 0)
                  return add_hook(lxc_conf, LXCHOOK_POSTSTOP, copy);
          else if (strcmp(key + 9, "clone") == 0)
                  return add_hook(lxc_conf, LXCHOOK_CLONE, copy);
          else if (strcmp(key + 9, "destroy") == 0)
                  return add_hook(lxc_conf, LXCHOOK_DESTROY, copy);

          free(copy);
          return -1;
  }
  ```

#### Do Not Use C99 Variable Length Arrays (VLA)

- They are made optional and there is no guarantee that future C standards
  will support them.

#### Use Standard libc Macros When Exiting

- libc provides `EXIT_FAILURE` and `EXIT_SUCCESS`. Use them whenever possible
  in the child of `fork()`ed process or when exiting from a `main()` function.

#### Use `goto`s

`goto`s are an essential language construct of C and are perfect to perform
cleanup operations or simplify the logic of functions. However, here are the
rules to use them:
- use descriptive `goto` labels.
  For example, if you know that this label is only used as an error path you
  should use something like `on_error` instead of `out` as label name.
- **only** jump downwards unless you are handling `EAGAIN` errors and want to
  avoid `do-while` constructs.
- An example of a good usage of `goto` is:
  ```
  static int set_config_idmaps(const char *key, const char *value,
                             struct lxc_conf *lxc_conf, void *data)
  {
          unsigned long hostid, nsid, range;
          char type;
          int ret;
          struct lxc_list *idmaplist = NULL;
          struct id_map *idmap = NULL;

          if (lxc_config_value_empty(value))
                  return lxc_clear_idmaps(lxc_conf);

          idmaplist = malloc(sizeof(*idmaplist));
          if (!idmaplist)
                  goto on_error;

          idmap = malloc(sizeof(*idmap));
          if (!idmap)
                  goto on_error;
          memset(idmap, 0, sizeof(*idmap));

          ret = parse_idmaps(value, &type, &nsid, &hostid, &range);
          if (ret < 0) {
                  ERROR("Failed to parse id mappings");
                  goto on_error;
          }

          INFO("Read uid map: type %c nsid %lu hostid %lu range %lu", type, nsid, hostid, range);
          if (type == 'u')
                  idmap->idtype = ID_TYPE_UID;
          else if (type == 'g')
                  idmap->idtype = ID_TYPE_GID;
          else
                  goto on_error;

          idmap->hostid = hostid;
          idmap->nsid = nsid;
          idmap->range = range;
          idmaplist->elem = idmap;
          lxc_list_add_tail(&lxc_conf->id_map, idmaplist);

          if (!lxc_conf->root_nsuid_map && idmap->idtype == ID_TYPE_UID)
                  if (idmap->nsid == 0)
                          lxc_conf->root_nsuid_map = idmap;


          if (!lxc_conf->root_nsgid_map && idmap->idtype == ID_TYPE_GID)
                  if (idmap->nsid == 0)
                          lxc_conf->root_nsgid_map = idmap;

          idmap = NULL;

          return 0;

  on_error:
          free(idmaplist);
          free(idmap);

          return -1;
  }
  ```

#### Use Booleans instead of integers

- When something can be conceptualized in a binary way use a boolean not
  an integer.

#### Cleanup Functions Must Handle The Object's Null Type And Being Passed Already Cleaned Up Objects

- If you implement a custom cleanup function to e.g. free a complex type
  you declared you must ensure that the object's null type is handled and
  treated as a NOOP. For example:
  ```
  void lxc_free_array(void **array, lxc_free_fn element_free_fn)
  {
          void **p;
          for (p = array; p && *p; p++)
                  element_free_fn(*p);
          free((void*)array);
  }
  ```
- Cleanup functions should also expect to be passed already cleaned up objects.
  One way to handle this cleanly is to initialize the cleaned up variable to
  a special value that signals the function that the element has already been
  freed on the next call. For example, the following function cleans up file
  descriptors and sets the already closed file descriptors to `-EBADF`. On the
  next call it can simply check whether the file descriptor is positive and
  move on if it isn't:
  ```
  static void lxc_put_attach_clone_payload(struct attach_clone_payload *p)
  {
          if (p->ipc_socket >= 0) {
                  shutdown(p->ipc_socket, SHUT_RDWR);
                  close(p->ipc_socket);
                  p->ipc_socket = -EBADF;
          }

          if (p->pty_fd >= 0) {
                  close(p->pty_fd);
                  p->pty_fd = -EBADF;
          }

          if (p->init_ctx) {
                  lxc_proc_put_context_info(p->init_ctx);
                  p->init_ctx = NULL;
          }
  }
  ```

### Cast to `(void)` When Intentionally Ignoring Return Values

- There are cases where you do not care about the return value of a function.
  Please cast the return value to `(void)` when doing so.
- Standard library functions or functions which are known to be ignored by
  default do not need to be cast to `(void)`. Classical candidates are
  `close()` and `fclose()`.
- A good example is:
  ```
  for (i = 0; hierarchies[i]; i++) {
          char *fullpath;
          char *path = hierarchies[i]->fullcgpath;

          ret = chowmod(path, destuid, nsgid, 0755);
          if (ret < 0)
                  return -1;

          /* failures to chown() these are inconvenient but not
           * detrimental we leave these owned by the container launcher,
           * so that container root can write to the files to attach.  we
           * chmod() them 664 so that container systemd can write to the
           * files (which systemd in wily insists on doing).
           */

          if (hierarchies[i]->version == cgroup_super_magic) {
                  fullpath = must_make_path(path, "tasks", null);
                  (void)chowmod(fullpath, destuid, nsgid, 0664);
                  free(fullpath);
          }

          fullpath = must_make_path(path, "cgroup.procs", null);
          (void)chowmod(fullpath, destuid, 0, 0664);
          free(fullpath);

          if (hierarchies[i]->version != cgroup2_super_magic)
                  continue;

          fullpath = must_make_path(path, "cgroup.subtree_control", null);
          (void)chowmod(fullpath, destuid, nsgid, 0664);
          free(fullpath);

          fullpath = must_make_path(path, "cgroup.threads", null);
          (void)chowmod(fullpath, destuid, nsgid, 0664);
          free(fullpath);
  }
  ```

#### Use `_exit()` in `fork()`ed Processes

- This has multiple reasons but the gist is:
  - `exit()` is not thread-safe
  - `exit()` in libc runs exit handlers which might interfer with the parents
    state

#### Use `for (;;)` instead of `while (1)` or `while (true)`

- Let's be honest, it is really the only sensible way to do this.

#### Use The Set Of Supported DCO Statements

- Signed-off-by: Random J Developer <random@developer.org>
  - You did write this code or have the right to contribute it to LXC.
- Acked-by: Random J Developer <random@developer.org>
  - You did read the code and think it is correct. This is usually only used by
    maintainers or developers that have made significant contributions and can
    vouch for the correctness of someone else's code.
- Reviewed-by: Random J Developer <random@developer.org>
  - You did review the code and vouch for its correctness, i.e. you'd be
    prepared to fix bugs it might cause. This is usually only used by
    maintainers or developers that have made significant contributions and can
    vouch for the correctness of someone else's code.
- Co-developed-by: Random J Developer <random@developer.org>
  - The code can not be reasonably attributed to a single developer, i.e.
    you worked on this together.
- Tested-by: Random J Developer <random@developer.org>
  - You verified that the code fixes a given bug or is behaving as advertised.
- Reported-by: Random J Developer <random@developer.org>
  - You found and reported the bug.
- Suggested-by: Random J Developer <random@developer.org>
  - You wrote the code but someone contributed the idea. This line is usually
    overlooked but it is a sign of good etiquette and coding ethics: if someone
    helped you solve a problem or had a clever idea do not silently claim it by
    slapping your Signed-off-by underneath. Be honest and add a Suggested-by.

#### Commit Message Outline

- You **must** stick to the 80chars limit especially in the title of the commit
  message.
- Please use English commit messages only.
- use meaningful commit messages.
- Use correct spelling and grammar.
  If you are not a native speaker and/or feel yourself struggling with this it
  is perfectly fine to point this out and there's no need to apologize. Usually
  developers will be happy to pull your branch and adopt the commit message.
- Please always use the affected file (without the file type suffix) or module
  as a prefix in the commit message.
- Examples of good commit messages are:
  ```
  commit b87243830e3b5e95fa31a17cf1bfebe55353bf13
  Author: Felix Abecassis <fabecassis@nvidia.com>
  Date:   Fri Feb 2 06:19:13 2018 -0800

      hooks: change the semantic of NVIDIA_VISIBLE_DEVICES=""

      With LXC, you can override the value of an environment variable to
      null, but you can't unset an existing variable.

      The NVIDIA hook was previously activated when NVIDIA_VISIBLE_DEVICES
      was set to null. As a result, it was not possible to disable the hook
      by overriding the environment variable in the configuration.

      The hook can now be disabled by setting NVIDIA_VISIBLE_DEVICES to
      null or to the new special value "void".

      Signed-off-by: Felix Abecassis <fabecassis@nvidia.com>


  commit d6337a5f9dc7311af168aa3d586fdf239f5a10d3
  Author: Christian Brauner <christian.brauner@ubuntu.com>
  Date:   Wed Jan 31 16:25:11 2018 +0100

      cgroups: get controllers on the unified hierarchy

      Signed-off-by: Christian Brauner <christian.brauner@ubuntu.com>

  ```