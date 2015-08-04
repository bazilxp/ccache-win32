# Why Not Use ccache shipped with Cygwin ? #

`cygwin1.dll` layer brings overheads, and we need install huge `Cygwin` distribution even if we just want use `ccache` under `MinGW` or `vxWorks`.

# ccache-win32.diff #

`ccache-win32.diff` was original from http://www.volny.cz/xnavara/ , http://www.volny.cz/xnavara/ also provide binary version of `ccache.exe`, but it will crash when compile certain source files.

And this piece of patch was aimed for CVS bleeding edge, not as stable as already released version 2.4, so I merged this patch to `ccache 2.4`, made a `ccache-win32 2.4` for win32.

`ccache-win32.diff` contents:
```
Index: ccache.c
===================================================================
RCS file: /cvsroot/ccache/ccache.c,v
retrieving revision 1.98
diff -u -p -r1.98 ccache.c
--- ccache.c	24 Nov 2005 21:54:09 -0000	1.98
+++ ccache.c	14 Jan 2006 10:04:44 -0000
@@ -491,11 +491,26 @@ static void from_cache(int first)
 		ret = 0;
 	} else {
 		unlink(output_file);
+#ifdef _WIN32
+                /* try to create hard link first if we were asked for it */
+		if (getenv("CCACHE_HARDLINK")) {
+			ret = CreateHardLinkA(output_file, hashname, NULL) ? 0 : -1;
+		} else {
+			ret = -1;
+		}
+
+		/* if that failed or the user hasn't asked for hard links,
+		   use regular copy. */
+		if (ret == -1) {
+			ret = copy_file(hashname, output_file);
+		}
+#else
 		if (getenv("CCACHE_HARDLINK")) {
 			ret = link(hashname, output_file);
 		} else {
 			ret = copy_file(hashname, output_file);
 		}
+#endif
 	}
 
 	/* the hash file might have been deleted by some external process */
@@ -569,7 +584,7 @@ static void find_compiler(int argc, char
 	if (strcmp(base, MYNAME) == 0) {
 		args_remove_first(orig_args);
 		free(base);
-		if (strchr(argv[1],'/')) {
+		if (strchr(argv[1],'/') || strchr(argv[1],'\\')) {
 			/* a full path was given */
 			return;
 		}
Index: ccache.h
===================================================================
RCS file: /cvsroot/ccache/ccache.h,v
retrieving revision 1.54
diff -u -p -r1.54 ccache.h
--- ccache.h	25 Jul 2005 07:05:46 -0000	1.54
+++ ccache.h	14 Jan 2006 10:01:16 -0000
@@ -8,8 +8,14 @@
 #include <errno.h>
 #include <sys/types.h>
 #include <sys/stat.h>
+#ifndef _WIN32
 #include <sys/wait.h>
 #include <sys/mman.h>
+#else
+#define _WIN32_WINNT 0x0500
+#include <windows.h>
+#include <shlobj.h>
+#endif
 #include <fcntl.h>
 #include <time.h>
 #include <string.h>
@@ -27,7 +33,11 @@
 #define STATUS_FATAL 4
 #define STATUS_NOCACHE 5
 
+#ifdef _WIN32
+#define MYNAME "ccache.exe"
+#else
 #define MYNAME "ccache"
+#endif
 
 #define LIMIT_MULTIPLE 0.8
 
Index: execute.c
===================================================================
RCS file: /cvsroot/ccache/execute.c,v
retrieving revision 1.10
diff -u -p -r1.10 execute.c
--- execute.c	6 Sep 2004 13:11:15 -0000	1.10
+++ execute.c	14 Jan 2006 09:59:16 -0000
@@ -18,6 +18,33 @@
 
 #include "ccache.h"
 
+#ifdef _WIN32
+static char *argvtos(char **argv)
+{
+	int i, len;
+	char *ptr, *str;
+
+	for (i = 0, len = 0; argv[i]; i++) {
+		len += strlen(argv[i]) + 3;
+	}
+
+	str = ptr = (char *)malloc(len + 1);
+	if (str == NULL)
+		return NULL;
+
+	for (i = 0; argv[i]; i++) {
+		len = strlen(argv[i]);
+		*ptr++ = '"';
+		memcpy(ptr, argv[i], len);
+		ptr += len;
+		*ptr++ = '"';
+		*ptr++ = ' ';
+	}
+	*ptr = 0;
+
+	return str;
+}
+#endif
 
 /*
   execute a compiler backend, capturing all output to the given paths
@@ -27,6 +54,55 @@ int execute(char **argv, 
 	    const char *path_stdout,
 	    const char *path_stderr)
 {
+#ifdef _WIN32
+	PROCESS_INFORMATION pinfo; 
+	STARTUPINFO sinfo;
+	BOOL ret; 
+	DWORD exitcode;
+	char *args;
+	HANDLE fd_out, fd_err;
+	SECURITY_ATTRIBUTES sa = {sizeof(SECURITY_ATTRIBUTES), NULL, TRUE};
+
+	fd_out = CreateFile(path_stdout, GENERIC_WRITE, 0, &sa, CREATE_ALWAYS,
+                            FILE_ATTRIBUTE_NORMAL, NULL);
+	if (fd_out == INVALID_HANDLE_VALUE) {
+		return STATUS_NOCACHE;
+	}
+
+	fd_err = CreateFile(path_stderr, GENERIC_WRITE, 0, &sa, CREATE_ALWAYS,
+                            FILE_ATTRIBUTE_NORMAL, NULL);
+	if (fd_err == INVALID_HANDLE_VALUE) {
+		return STATUS_NOCACHE;
+	}
+   
+	ZeroMemory(&pinfo, sizeof(PROCESS_INFORMATION));
+	ZeroMemory(&sinfo, sizeof(STARTUPINFO));
+
+	sinfo.cb = sizeof(STARTUPINFO); 
+	sinfo.hStdError = fd_err;
+	sinfo.hStdOutput = fd_out;
+	sinfo.hStdInput = GetStdHandle(STD_INPUT_HANDLE);
+	sinfo.dwFlags |= STARTF_USESTDHANDLES;
+ 
+	args = argvtos(argv);
+
+	ret = CreateProcessA(argv[0], args, NULL, NULL, TRUE, 0, NULL, NULL,
+	                     &sinfo, &pinfo);
+
+	free(args);
+	CloseHandle(fd_out);
+	CloseHandle(fd_err);
+
+	if (ret == 0)
+		return -1;
+
+	WaitForSingleObject(pinfo.hProcess, INFINITE);
+	GetExitCodeProcess(pinfo.hProcess, &exitcode);
+	CloseHandle(pinfo.hProcess);
+	CloseHandle(pinfo.hThread);
+
+	return exitcode;
+#else
 	pid_t pid;
 	int status;
 
@@ -64,6 +140,7 @@ int execute(char **argv, 
 	}
 
 	return WEXITSTATUS(status);
+#endif
 }
 
 
@@ -72,6 +149,18 @@ int execute(char **argv, 
 */
 char *find_executable(const char *name, const char *exclude_name)
 {
+#if _WIN32
+	DWORD ret;
+	char namebuf[MAX_PATH];
+
+        ret = SearchPathA(getenv("CCACHE_PATH"), name, ".exe",
+                          sizeof(namebuf), namebuf, NULL);
+        if (ret != 0) {
+        	return x_strdup(namebuf);
+        }
+
+        return NULL;
+#else
 	char *path;
 	char *tok;
 	struct stat st1, st2;
@@ -126,4 +215,5 @@ char *find_executable(const char *name, 
 	}
 
 	return NULL;
+#endif
 }
Index: unify.c
===================================================================
RCS file: /cvsroot/ccache/unify.c,v
retrieving revision 1.8
diff -u -p -r1.8 unify.c
--- unify.c	6 Sep 2004 12:24:05 -0000	1.8
+++ unify.c	13 Jan 2006 21:52:18 -0000
@@ -238,6 +238,39 @@ static void unify(unsigned char *p, size
 */
 int unify_hash(const char *fname)
 {
+#ifdef _WIN32
+	HANDLE file;
+	HANDLE section;
+	MEMORY_BASIC_INFORMATION minfo;
+	char *map;
+
+	file = CreateFileA(fname, GENERIC_READ, FILE_SHARE_READ, NULL,
+	                   OPEN_EXISTING, 0, NULL);
+	if (file == INVALID_HANDLE_VALUE)
+		return -1;
+
+	section = CreateFileMappingA(file, NULL, PAGE_READONLY, 0, 0, NULL);
+	CloseHandle(file);
+	if (section == NULL)
+		return -1;
+
+	map = MapViewOfFile(section, FILE_MAP_READ, 0, 0, 0);
+	CloseHandle(section);
+	if (map == NULL)
+		return -1;
+
+        if (VirtualQuery(map, &minfo, sizeof(minfo)) != sizeof(minfo)) {
+		UnmapViewOfFile(map);
+		return -1;
+        }
+
+	/* pass it through the unifier */
+	unify((unsigned char *)map, minfo.RegionSize);
+
+	UnmapViewOfFile(map);
+
+	return 0;
+#else
 	int fd;
 	struct stat st;	
 	char *map;
@@ -265,5 +298,6 @@ int unify_hash(const char *fname)
 	munmap(map, st.st_size);
 
 	return 0;
+#endif
 }
 
Index: util.c
===================================================================
RCS file: /cvsroot/ccache/util.c,v
retrieving revision 1.36
diff -u -p -r1.36 util.c
--- util.c	18 May 2005 04:32:31 -0000	1.36
+++ util.c	14 Jan 2006 09:59:38 -0000
@@ -57,6 +57,15 @@ void copy_fd(int fd_in, int fd_out)
 	}
 }
 
+#ifndef HAVE_MKSTEMP
+/* cheap and nasty mkstemp replacement */
+int mkstemp(char *template)
+{
+	mktemp(template);
+	return open(template, O_RDWR | O_CREAT | O_EXCL | O_BINARY, 0600);
+}
+#endif
+
 /* copy a file - used when hard links don't work 
    the copy is done via a temporary file and atomic rename
 */
@@ -96,9 +105,11 @@ int copy_file(const char *src, const cha
 	close(fd1);
 
 	/* get perms right on the tmp file */
+#ifndef _WIN32
 	mask = umask(0);
 	fchmod(fd2, 0666 & ~mask);
 	umask(mask);
+#endif
 
 	/* the close can fail on NFS if out of space */
 	if (close(fd2) == -1) {
@@ -132,9 +143,15 @@ int create_dir(const char *dir)
 		errno = ENOTDIR;
 		return 1;
 	}
+#ifdef _WIN32
+	if (mkdir(dir) != 0 && errno != EEXIST) {
+		return 1;
+	}
+#else
 	if (mkdir(dir, 0777) != 0 && errno != EEXIST) {
 		return 1;
 	}
+#endif
 	return 0;
 }
 
@@ -221,7 +238,11 @@ void traverse(const char *dir, void (*fn
 		if (strlen(de->d_name) == 0) continue;
 
 		x_asprintf(&fname, "%s/%s", dir, de->d_name);
+#ifdef _WIN32
+		if (stat(fname, &st)) {
+#else
 		if (lstat(fname, &st)) {
+#endif
 			if (errno != ENOENT) {
 				perror(fname);
 			}
@@ -244,11 +265,16 @@ void traverse(const char *dir, void (*fn
 /* return the base name of a file - caller frees */
 char *str_basename(const char *s)
 {
-	char *p = strrchr(s, '/');
+	char *p;
+	
+	p = strrchr(s, '/');
 	if (p) {
-		return x_strdup(p+1);
-	} 
-
+		s = (p+1);
+	}
+	p = strrchr(s, '\\');
+	if (p) {
+		s = (p+1);
+	}
 	return x_strdup(s);
 }
 
@@ -261,11 +287,16 @@ char *dirname(char *s)
 	if (p) {
 		*p = 0;
 	} 
+	p = strrchr(s, '\\');
+	if (p) {
+		*p = 0;
+	} 
 	return s;
 }
 
 int lock_fd(int fd)
 {
+#ifndef _WIN32
 	struct flock fl;
 	int ret;
 
@@ -281,17 +312,24 @@ int lock_fd(int fd)
 		ret = fcntl(fd, F_SETLKW, &fl);
 	} while (ret == -1 && errno == EINTR);
 	return ret;
+#else
+	return 0;
+#endif
 }
 
 /* return size on disk of a file */
 size_t file_size(struct stat *st)
 {
+#ifdef _WIN32
+	return (st->st_size + 1023) & ~1023;
+#else
 	size_t size = st->st_blocks * 512;
 	if ((size_t)st->st_size > size) {
 		/* probably a broken stat() call ... */
 		size = (st->st_size + 1023) & ~1023;
 	}
 	return size;
+#endif
 }
 
 
@@ -353,6 +391,17 @@ size_t value_units(const char *s)
 */
 char *x_realpath(const char *path)
 {
+#ifdef _WIN32
+	char namebuf[MAX_PATH];
+	DWORD ret;
+
+	ret = GetFullPathNameA(path, sizeof(namebuf), namebuf, NULL);
+	if (ret == 0 || ret >= sizeof(namebuf)) {
+		return NULL;
+	}
+
+	return x_strdup(namebuf);
+#else
 	int maxlen;
 	char *ret, *p;
 #ifdef PATH_MAX
@@ -388,6 +437,7 @@ char *x_realpath(const char *path)
 	}
 	free(ret);
 	return NULL;
+#endif
 }
 
 /* a getcwd that will returns an allocated buffer */
@@ -408,16 +458,6 @@ char *gnu_getcwd(void)
 	}
 }
 
-#ifndef HAVE_MKSTEMP
-/* cheap and nasty mkstemp replacement */
-int mkstemp(char *template)
-{
-	mktemp(template);
-	return open(template, O_RDWR | O_CREAT | O_EXCL | O_BINARY, 0600);
-}
-#endif
-
-
 /* create an empty file */
 int create_empty_file(const char *fname)
 {
@@ -436,6 +476,24 @@ int create_empty_file(const char *fname)
 */
 const char *get_home_directory(void)
 {
+#ifdef _WIN32
+	static char home_path[MAX_PATH] = {0};
+	HRESULT ret;
+
+	/* we already have the path */
+	if (home_path[0] != 0) {
+		return home_path;
+	}
+
+	/* get the path to "Application Data" folder */
+	ret = SHGetFolderPathA(NULL, CSIDL_APPDATA | CSIDL_FLAG_CREATE, NULL, 0, home_path);
+	if (SUCCEEDED(ret)) {
+		return home_path;
+	}
+
+	fprintf(stderr, "ccache: Unable to determine home directory\n");
+	return NULL;
+#else
 	const char *p = getenv("HOME");
 	if (p) {
 		return p;
@@ -448,7 +506,8 @@ const char *get_home_directory(void)
 		}
 	}
 #endif
-	fprintf(stderr, "ccache: Unable to determine home directory");
+	fprintf(stderr, "ccache: Unable to determine home directory\n");
 	return NULL;
+#endif
 }
 
```