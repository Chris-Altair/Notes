文件分隔符，屏蔽操作系统差异：File.separator

file.deleteOnExit() 删除文件声明，并不立刻删除，jvm停止时才删除

文件流最后必须要close()，若没有close，可能出现**文件句柄被占用**的情况，这样其他进程就无法修改该文件了

​	file1.delete())返回false，文件存在但删不掉,发现被java占用经测试发现文件流未关闭，结论手动创建文件流一定要close

文件流：https://www.zhihu.com/question/67535292

java比较2个文件是否相同，通过文件内容的md5判断，相同则相当大概率相同，不相同则一定不同

```java
/**
     * 获取一个文件的md5值(可处理大文件)
     * @return md5 value
     */
    public static String getMD5(File file) {
        FileInputStream fileInputStream = null;
        try {
            MessageDigest MD5 = MessageDigest.getInstance("MD5");
            fileInputStream = new FileInputStream(file);
            byte[] buffer = new byte[8192];//1024*8 有讲究的
            int length;
            while ((length = fileInputStream.read(buffer)) != -1) {
                MD5.update(buffer, 0, length);
            }
            return new String(Hex.encodeHex(MD5.digest()));
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        } finally {
            try {
                if (fileInputStream != null){
                    fileInputStream.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```



java压缩文件操作

1. new ZipOutputStream(new FileOutputStream(zippath))
2. zos.putNextEntry(new ZipEntry(file.getName())) //按要压缩的文件循环
3. zos.write //按要压缩的文件循环

```java
public static final int BUFFER_SIZE = 1024 * 8;

    public static void createZip(String zipName, String sourcePath, String zipPath) {
        File sourceDir = new File(sourcePath);
        if (!sourceDir.exists()) {
            throw new RuntimeException("源目录不存在");
        }
        if (!sourceDir.isDirectory()) {
            throw new RuntimeException("源路径非目录");
        }
        File zipDir = new File(zipPath);
        if (!zipDir.exists()) {
            zipDir.mkdirs();
        }
        File zip = new File(zipPath + zipName + ".zip");
        if (zip.exists()) {
            throw new RuntimeException("zip文件已存在");
        }
        FileOutputStream fos = null;
        ZipOutputStream zos = null;
        FileInputStream fis = null;
        BufferedInputStream bis = null;
        try {
            File[] sourceFiles = sourceDir.listFiles();
            if (sourceFiles == null || sourceFiles.length < 1) {
                throw new RuntimeException("源文件夹为空");
            }
            fos = new FileOutputStream(zip);
            zos = new ZipOutputStream(fos);
            zos.setComment("压缩test");
            byte[] buffer = new byte[BUFFER_SIZE];
            for (File sourceFile : sourceFiles) {
                ZipEntry zipEntry = new ZipEntry(sourceFile.getName());
                zos.putNextEntry(zipEntry);
                System.out.println(sourceFile.getName());
                fis = new FileInputStream(sourceFile);
                bis = new BufferedInputStream(fis, BUFFER_SIZE);
                int read = 0;
                while ((read = bis.read(buffer, 0, BUFFER_SIZE)) != -1) {
                    zos.write(buffer, 0, read);
                }
                //先关闭外面，再关闭里面
                bis.close();
                fis.close();
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                //先关闭外面，再关闭里面
                if (zos != null) {
                    zos.close();
                }
                if (fos != null) {
                    fos.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

