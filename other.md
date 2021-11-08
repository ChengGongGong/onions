# 1. 读取url中的压缩文件的.json文件

private Map<String,String> readData(String urlStr) {

        URL url = null;
        HttpURLConnection conn = null;
        //得到输入流
        Map<String, String> jsonMap = new HashMap<>();
        try {
            url = new URL(urlStr);
            conn = (HttpURLConnection) url.openConnection();
            //设置超时间为3秒
            conn.setConnectTimeout(3 * 1000);
            conn.setRequestProperty("Charset","UTF-8");
        } catch (IOException e ) {
           e.printStackTrace();
        }
        if(conn==null){
          return jsonMap;
        }
        try(InputStream inputStream= conn.getInputStream();
                ZipInputStream zin=new ZipInputStream(inputStream, StandardCharsets.UTF_8);
                BufferedInputStream bs=new BufferedInputStream(zin);
                ) {
            byte[] bytes = null;

            ZipEntry ze;
            //循环读取压缩包里面的文件
            while ((ze = zin.getNextEntry()) != null) {
                StringBuilder orginJson = new StringBuilder();
                if (ze.toString().endsWith(".json")) {
                    //读取每个文件的字节，并放进数组
                    bytes = new byte[(int) ze.getSize()];
                    bs.read(bytes, 0, (int) ze.getSize());
                    //将文件转成流
                    InputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
                    BufferedReader br = new BufferedReader(new InputStreamReader(byteArrayInputStream,
                                                                                 StandardCharsets.UTF_8));
                    //读取文件里面的内容
                    String line;
                    while ((line = br.readLine()) != null) {
                        orginJson.append(line);
                    }
                    //关闭流
                    br.close();
                    String name = new String(ze.getName().replace(".json", ""));
                    jsonMap.put(name, orginJson.toString());
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return jsonMap;
    }
    
 # 2. 对集合进行深拷贝,对泛型类进行序列化(实现Serializable)
 public <T> List<T> deepCopy(List<T> src)  {

        List<T> dest=new ArrayList<>();
        ByteArrayOutputStream byteOut = new ByteArrayOutputStream();
        try(ObjectOutputStream out=new ObjectOutputStream(byteOut)){
            out.writeObject(src);
            ByteArrayInputStream byteIn=new ByteArrayInputStream(byteOut.toByteArray());
            try(ObjectInputStream in=new ObjectInputStream(byteIn)){
                dest=(List<T>)in.readObject();
            }catch (ClassNotFoundException e){
                e.printStackTrace();
            }
        }catch (IOException e){
            e.printStackTrace();
        }
        return dest;
    }

        
# 3.属性拷贝
   
 主要包括以下几种类型：
    
    org.apache.commons.beanutils.PropertyUtils#copyProperties：
    1.基本类型和包装类型会自动转换
    2.方法名称相同，返回值类型和参数类型不同，复制失败，会报错
    3.只支持类的修饰符 public，如果是default 则直接不会进行转换（注意内部类复制也要加public）
    
    org.apache.commons.beanutils.BeanUtils#_copyProperties：
    1.基本类型和包装类型会自动转换
    2.方法名称相同，返回值类型和参数类型不同，不复制，不报错
    3.只支持类的修饰符 public，如果是default 则直接不会进行转换（注意内部类复制也要加public）
    
    org.springframework.beans.BeanUtils#copyProperties：
    1.基本类型和包装类型会自动转换, 方法名称相同，返回值类型和参数类型不同，不进行复制，也不报错；
    2.支持指定忽略某些属性不复制；
    3.支持类的修饰符 default 、 public
    
   org.springframework.cglib.beans.BeanCopier#create:
    支持两种方式，一种是不使用Converter的方式，仅对两个bean间属性名和类型完全相同的变量进行拷贝。另一种则引入Converter，可以对某些特定属性值进行特殊操作
    例如：
    
    @Slf4j
    public class BeanUtil {

        private static final Map<String, BeanCopier> COPIER_MAP = new ConcurrentHashMap<>(16);

        public static <T, V> V copyFrom(T src, Class<V> target){
            StringBuilder sb = new StringBuilder();
            sb.append(src.getClass().getName()).append("-").append(target.getName());
            BeanCopier beanCopier;

            if (!COPIER_MAP.containsKey(sb.toString())) {
                beanCopier = BeanCopier.create(src.getClass(), target, false);
                COPIER_MAP.put(sb.toString(), beanCopier);
            }
            beanCopier = COPIER_MAP.get(sb.toString());
            V result;

            try {
                result = target.getDeclaredConstructor().newInstance();
                beanCopier.copy(src, result, null);
                return result;
            } catch (Exception e) {
                log.error("Bean copy fail error", e);
                return null;
            }
        }
    }
# 4. 上传文件到云存储

## 1.引入aws相关pom

<dependency>

            <groupId>com.amazonaws</groupId>
            <artifactId>aws-java-sdk</artifactId>
            <version>1.11.787</version>
            <exclusions>
                <exclusion>
                    <artifactId>jackson-databind</artifactId>
                    <groupId>com.fasterxml.jackson.core</groupId>
                </exclusion>
            </exclusions>
            
</dependency>

## 2.示例代码如下

 public static void main(String[] args) throws IOException, InterruptedException {
 
        MsgVO msgvo=new MsgVO();

        String  accessKeyId = "";
        String accessKeySecret = "";
        String endPoint = "";
        String region  = "";

        String bucketName = "";
        String key = "xxx.html";
        msgvo.setAccessKeyId(accessKeyId);
        msgvo.setAccessKeySecret(accessKeySecret);
        msgvo.setEndPoint(endPoint);
        msgvo.setRegion(region);
        msgvo.setBucketName(bucketName);
        msgvo.setKey(key);
        msgvo.setFile(createSampleFile());

        upload(msgvo, new FileInputStream(msgvo.getFile()),Constant.CONTENT_TYPE_HTML);

    }
    private static File createSampleFile() throws IOException {
        File file = new File("C:\\Users\\xxx.html");
        return file;
    }
    public static void upload(MsgVO msgVO, InputStream inputStream, String contentType) {

        AWSCredentials awsCredentials = new BasicAWSCredentials(msgVO.getAccessKeyId(),
                                                                msgVO.getAccessKeySecret());
        AWSCredentialsProvider awsCredentialsProvider = new AWSStaticCredentialsProvider(
                awsCredentials);
        ClientConfiguration clientConfiguration = new ClientConfiguration()
                .withProtocol(Protocol.HTTP).withSignerOverride("S3SignerType");
        AmazonS3 s3 = AmazonS3ClientBuilder.standard()
                                           .withCredentials(awsCredentialsProvider)
                                           .withPathStyleAccessEnabled(true)
                                           .withClientConfiguration(clientConfiguration)
                                           .withEndpointConfiguration(
                                                   new AwsClientBuilder.EndpointConfiguration(
                                                           msgVO.getEndPoint(),
                                                           msgVO.getRegion())).build();
        log.info("Putting object to:{},key:{}", msgVO.getBucketName(),msgVO.getKey());
        long partSize = 4 * 1024 * 1024L;
        TransferManagerBuilder builder = TransferManagerBuilder.standard()
                                                               .withS3Client(s3)
                                                               .withMultipartUploadThreshold(3 * partSize)
                                                               .withMinimumUploadPartSize(partSize);
        TransferManager manager = builder.build();
        ObjectMetadata objectMetadata = new ObjectMetadata();
        objectMetadata.setContentType(contentType);
        try {
            Upload upload = manager.upload(msgVO.getBucketName(), msgVO.getKey(), inputStream, objectMetadata);
            UploadResult result = upload.waitForUploadResult();
            log.info("result:{}", result);
        } catch (Exception e) {
            log.error("upload error", e);
        }
    }

    static class Constant {

        public static final String CONTENT_TYPE_HTML = "text/html";
        public static final String CONTENT_TYPE_PNG = "image/png";
        public static final String CONTENT_TYPE_JPG = "image/jpeg";
        public static final String CONTENT_TYPE_GIF = "image/gif";
        public static final String CONTENT_TYPE_CSS = "text/css";
        public static final String CONTENT_TYPE_JS = "application/javascript";
    }
  
    public class MsgVO {

        private String accessKeyId ;
    
        private String accessKeySecret ;
    
        private String endPoint ;
    
        private String region  ;

        private String bucketName;
    
        private String key;

        private String downloadUrl;

        private String cdnStatus;

        private File file;
        
     }
