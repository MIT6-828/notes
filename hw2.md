```
void
runcmd(struct cmd *cmd)
{
  int p[2], r;
  struct execcmd *ecmd;
  struct pipecmd *pcmd;
  struct redircmd *rcmd;

  if(cmd == 0)
    _exit(0);

  switch(cmd->type){
  default:
    fprintf(stderr, "unknown runcmd\n");
    _exit(-1);

  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      _exit(0);
    if (execv(ecmd->argv[0], ecmd->argv) < 0) {
        perror("exec failed");
    }
    break;

  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;

    if (cmd->type == '>') close(1);
    else close(0);
    if (open(rcmd->file, rcmd->flags, 0666) < 0) {
        perror("open failed");
        break;
    }
    runcmd(rcmd->cmd);
    break;

  case '|':
    pcmd = (struct pipecmd*)cmd;

    int fd[2];
    if (pipe(fd) < 0) {
        perror("pipe failed");
        break;
    }
    if (fork1()) { // parent process closes the read end of the pipe and use
                   // the write end of the pipe as stdout
        close(fd[0]);
        close(1);
                dup(fd[1]);
        runcmd(pcmd->left);
    } else {       // child process closes the write end of the pipe and use
                   // the read end of the pipe as stdin
        close(fd[1]);
        close(0);
        dup(fd[0]);
        runcmd(pcmd->right);
    }
    break;
  }
  _exit(0);
}
```
