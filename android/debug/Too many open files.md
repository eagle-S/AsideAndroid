## Too many open files

## lsof 

https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/lsof.html

查看进程<pid>使用的文件句柄:
方法一：lsof -p <pid> | grep <pid>
方法二：ll /proc/<pid>/fd | busybox wc -l

查看进程<pid>文件句柄最大值
cat /proc/<pid>/limits | grep "Max open files"


Android4.4 源码LocalSocket泄漏

泄漏场景，使用serverSocket.accept()获取的LocalSocket对象在调用close后未真正关闭文件句柄

    LocalServerSocket serverSocket;

    try {
        serverSocket = new LocalServerSocket("localsocket_test");
    } catch (IOException e) {
        Log.e(TAG, "AudioTrackThread exception", e);
        return;
    }
    
    LocalSocket socket = null;
    try {
        socket = serverSocket.accept();
        din = new DataInputStream(socket.getInputStream());
        if (socket != null) {
            String s1 = din.readLine();//读入传来的字符串
           
        ............           
        }
    } catch (IOException e) {
        Log.e(TAG, "IOException", e);
    } finally {
        if (socket != null) {
            try {
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    
解决方法：参考Android5.0  LocalSocketImpl#accept

    protected void accept(LocalSocketImpl s) throws IOException
    {
        if (fd == null) {
            throw new IOException("socket not created");
        }

        s.fd = accept(fd, s);
        
        // 增加代码
        s.mFdCreatedInternally = true;
    }


cat /proc/sys/fs/file-max 

cat /proc/sys/fs/file-nr

