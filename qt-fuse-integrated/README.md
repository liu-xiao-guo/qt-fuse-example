# todo

this code is currently not working, and i stopped working on it for the time being as i don't have resources to go on without help.

## implementation using QFile
- interprete the multithreaded variable
 - implement multithreading like fuse_loop_mt()
 - implement splice
- add error handling and remove all hacky code parts
- verify that all these methods work well:
 - ctrl+c
 - 'fusermount -u dst'
 - 'kill 123'
- update this documentation, see the README from the qt-fuse direcotry
- REWRITE examplefs.cc::Read_buf

## implementation using QLocalSocket
see commit cf2747d790db0688b868141d09892c81d6fb3c91 where i got the idea behind qt-fuse-intrgrated partially working using a QLocalSocket. 
the current implementation based on a QFile does not work at all!

since /dev/fuse is a character device and not a socket it wonders me why this code works in the first place ;-)

# what is this?
this is a Qt4 example on how to integrate the FUSE eventloop (fuse_loop and/or fuse_loop_mt) into the
qcoreapplication's eventloop.

# what is currently implemented
- in QFuse.cpp a QLocalSocket is used to listen to /dev/fuse stuff, input from there is read and then sent using:
  fuse_session_process_buf(se, &fbuf, ch);
- implementes the concept of fuse_loop (singlethreading)

http://sourceforge.net/mailarchive/forum.php?thread_name=87eixgzgkc.fsf%40frosties.localdomain&forum_name=fuse-devel

see also
http://www.youblisher.com/files/publications/6/31627/pdf.pdf

==== Goswin ====
Easier to break up the fuse loop and have QT4 watch the fuse file
descripor for input in its main loop. Whenever there is some input you
call the process function of fuse. You have to look at the fuse main
loop to see how to use the lowlevel functions but it is not that hard.

That also means you have to use the lower level functions to create
the session and channel. So you have to look at fuse_main_init() for
the details as well.

qknight comments:
- fuse_main_init() does not exist
- did i use the fuse_lowlevel functions?


================ Joachim ================================================================
anatomy of the event loop spawned by FUSE:
 - a pipe is opened and read from: '/dev/fuse' (see 
================ lib/mount.c ============================================================
      417:    const char *devname = "/dev/fuse";
      449:    fd = open(devname, O_RDWR | O_CLOEXEC);
      455:                            devname, strerror(errno));
      470:                    strlen(devname) + 32);
      484:           mo->fsname ? mo->fsname : (mo->subtype ? mo->subtype : devname));
   )
 - the commands are serialized using 'struct fuse_in_header' the process_buf function callback

================ long story: ============================================================
 - the eventloop of FUSE is implemented in fuse_loop or fuse_loop_mt
 - this blocking loop is normally started from inside a new (p)thread
 - fuse_loop(..) works like this:
       int fuse_session_loop(struct fuse_session *se)
       {
              int res = 0;
              struct fuse_chan *ch = fuse_session_next_chan(se, NULL);
              size_t bufsize = fuse_chan_bufsize(ch);
              char *buf = (char *) malloc(bufsize);
              if (!buf) {
                     fprintf(stderr, "fuse: failed to allocate read buffer\n");
                     return -1;
              }

              while (!fuse_session_exited(se)) {
                     struct fuse_chan *tmpch = ch;
                     struct fuse_buf fbuf = {
                            .mem = buf,
                            .size = bufsize,
                     };

                     res = fuse_session_receive_buf(se, &fbuf, &tmpch);

                     if (res == -EINTR)
                            continue;
                     if (res <= 0)
                            break;

                     fuse_session_process_buf(se, &fbuf, tmpch);
              }

              free(buf);
              fuse_session_reset(se);
              return res < 0 ? -1 : 0;
       }
  - the res = fuse_session_receive_buf(se, &fbuf, &tmpch); internally uses a receive_buf callback which is
    declared in the fuse_session on se->receive_buf is set in:
================ lib/fuse_lowlevel.c ==============================================================
    2812:   se->receive_buf = fuse_ll_receive_buf;
    
  - in fuse_ll_receive_buf there are two implementations: a fallback one and a splice one
  - 'man 2 splice' helps to understand splice. in a nutshell:
      splice()  moves  data  between two file descriptors without copying between kernel address space and user address space.

  - res = fuse_chan_recv(chp, buf->mem, bufsize);
    calls this:
    lib/fuse_kern_chan.c
    87: struct fuse_chan_ops op = {
            .receive = fuse_kern_chan_receive,
            .send = fuse_kern_chan_send,
            .destroy = fuse_kern_chan_destroy,
        };
================ fuse_kern_chan.c: ================================================================
      static int fuse_kern_chan_receive(struct fuse_chan **chp, char *buf, size_t size)
      {
          struct fuse_chan *ch = *chp;
          int err;
          ssize_t res;
          struct fuse_session *se = fuse_chan_session(ch);
          assert(se != NULL);

      restart:
          res = read(fuse_chan_fd(ch), buf, size);
          err = errno;

          if (fuse_session_exited(se))
            return 0;
          if (res == -1) {
          /* ENOENT means the operation was interrupted, it's safe
            to restart */
          if (err == ENOENT)
              goto restart;

          if (err == ENODEV) {
              fuse_session_exit(se);
              return 0;
          }
          /* Errors occurring during normal operation: EINTR (read
            interrupted), EAGAIN (nonblocking I/O), ENODEV (filesystem
            umounted) */
          if (err != EINTR && err != EAGAIN)
              perror("fuse: reading device");
          return -err;
          }
          if ((size_t) res < sizeof(struct fuse_in_header)) {
          fprintf(stderr, "short read on fuse device\n");
          return -EIO;
          }
          return res;
      }
================================================================================================        

