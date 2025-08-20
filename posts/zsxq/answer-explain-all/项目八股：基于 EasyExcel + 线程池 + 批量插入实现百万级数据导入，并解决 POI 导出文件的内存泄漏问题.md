
背景
---

项目中有一个数据迁移，原来的数据存储在旧的系统，现在系统做了重构，需要迁移到新的系统中，老系统的数据被加工到Excel中了，需要基于Excel实现文件的导入，同时需要避免内存溢出以及性能太低的问题。

问题分析
----

* 1.  
  **内存溢出问题**

*  
  百万级别的数据量的 Excel 文件会非常大，如果全都加载到内存中，可能会导致内存溢出问题。

* 1.  
  **性能问题**

*  
  百万级别数据从 Excel 读取并且插入到数据中， 可能会很慢，所以需要考虑性能来问题

* 1.  
  **错误处理**

*  
  在文件的读取以及导入过程中，可能会出现各种各样的问题，我们要妥善解决好这些问题。

内存溢出问题
------

百万级别数据量，如果一次性读取到内存中，肯定是不现实的，那么最好的办法就是**基于流式读取的方式进行分批处理**。

在技术选型上面，我们选择使用 EasyExcel，特别是针对大数据量和复杂 Excel 文件的处理进行优化。在解析 Excel 的时候，EasyExcel 不会一次性将 Excel 一次性全部加载到内存中，而是从磁盘上面一行行进行读取，逐步进行解析。

性能问题
----

百万级别的数据量，如果采用单线程进行读取的话，性能会非常非常 low ！！！

所以，针对这个问题，我们就需要使用**多线程。**

使用多线程主要涉及到两个场景：

*  
多线程读取  
*  
  多线程实现数据插入

这里就涉及到一个生产者---消费者的模式，通过使用多个线程进行读取，然后多个线程实现插入，从而最大限度地提升整体的性能来。

数据的插入，除了借助多线程以外，还可以同时使用**数据库的批量插入的功能，** 从而提高插入的速度。

错误处理
----

在文件的读取和数据库写入的过程中，需要解决各种各样的问题，比如数据格式错误、数据不一致、重复数据等问题。

所以我们需要分两步进行一个处理：

* 1.  
对数据进行检查，在开始插入之前将数据的格式问题提前检查好  
* 2.  
  在插入过程中，需要对异常进行处理

对于异常处理，我们的处理方式有很多，可以进行日志记录，也可以选择进行事务回滚，这个主要根据实际情况进行选择，一般情况下是不建议进行回滚的，直接自动重试，如果重试之后还是不行，则记录日志，再记录日志然后重新插入即可。

并且在这个过程中，需要考虑一下数据重复的问题，需要在 Excel 中某几个字段设置成数据库唯一约束，然后遇到数据冲突的情况，可以先进行处理，处理的方式可以是覆盖、跳过以及报错，这个根据业务的实际情况来确定，一般可以使用跳过+打印日志实现。

技术选型
----

在大文件的读取方面，EasyExcel更合适，因为他不会像POI一样耗内存，可以大大的减少内存占用。因为他并不会一次性把整个Excel都加载到内存中，而是逐行读取的。

同时考虑使用多线程来读取，这里就需要用到线程池的技术，直接用ExecutorService就行了。

因为还涉及到数据的批量写入，需要依赖mybatis或者mybatis-plus。

整体方案
----

使用 EasyExcel 实现 Excel 文件的读取，因为他不会一次性将整个 Excel 加载到内存中，是采取逐行读取的方式，并且为了提高并发性能，我们可以将数据先分散到不同的 sheet 中，然后借助线程池，多线程同时读取不同的sheet，在读取的过程中，借助 EasyExcel 的 ReadListener 进行处理。

在处理的过程中，我们并不会每一条数据都操作一次数据库，这样对于数据库的压力太大了，一般我们会设置一个批次，如 2000 条，我们从 Excel 中读取的数据暂时存储到内存中，这里可以使用 List 实现，在读取 2000 条数据，就可以执行一次数据的批量插入，这里可以使用 MyBatis 的批量插入以及 MP 的批量插入功能实现。

MyBatis 实现批量通过插入：<https://juejin.cn/post/7016691244973686820>


然后在这个过程中，需要考虑一些并发的问题，所以我们这里会使用一些线程安全的队列，比如 **ConcurrentLinkedQueue**

然后在经过验证以后，读取 100 w 的 Excel 并且插入数据之后，总耗时在 100 秒左右，用时不超过 2 分钟。

具体实现
----

为了提高并发处理能力，我们将百万级别的数据放到同一个 Excel 的不同 sheet 中， 然后通过使用 EasyExcel 并发读取这些 sheet。

EasyExcel 提供了 ReadListener 接口，允许在读取每一批数据之后自定义进行处理，我们可以基于这个功能来实现文件的分批读取。

### 依赖添加

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependencies>
    <!-- EasyExcel -->
    <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>最新的版本号</version>
    </dependency>

    <!-- 数据库连接和线程池 -->
    <dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>最新版本号</version>
    </dependency>
    <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
    </dependency>
    </dependencies>
              
### 代码实现

然后实现并发读取多个 sheet 代码：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Service
    public class ExcelImporterService {

        @Autowired
        private MyDataService myDataService;

        public void doImport() {
            // Excel文件的路径
            String filePath = "users/linqi/workspace/excel/test.xlsx";

            // 需要读取的sheet数量
            int numberOfSheets = 20;

            // 创建一个固定大小的线程池，大小与sheet数量相同
            ExecutorService executor = Executors.newFixedThreadPool(numberOfSheets);

            // 遍历所有sheets
            for (int sheetNo = 0; sheetNo < numberOfSheets; sheetNo++) {
                // 在Java lambda表达式中使用的变量需要是final
                int finalSheetNo = sheetNo;

                // 向线程池提交一个任务
                executor.submit(() -> {
                    // 使用EasyExcel读取指定的sheet
                    EasyExcel.read(filePath, MyDataModel.class, new MyDataModelListener(myDataService))
                    .sheet(finalSheetNo) // 指定sheet号
                    .doRead(); // 开始读取操作
                });
            }

            // 启动线程池的关闭序列
            executor.shutdown();

            // 等待所有任务完成，或者在等待超时前被中断
            try {
                executor.awaitTermination(Long.MAX_VALUE, TimeUnit.NANOSECONDS);
            } catch (InterruptedException e) {
                // 如果等待过程中线程被中断，打印异常信息
                e.printStackTrace();
            }
        }
    }
              
这段代码通过创建一个固定大小的线程池来并发读取一个包含多个sheets的Excel文件。每个sheet的读取作为一个单独的任务提交给线程池。

我们在代码中用了一个MyDataModelListener，这个类是ReadListener的一个实现类。当EasyExcel读取每一行数据时，它会自动调用我们传入的这个ReadListener实例的invoke方法。在这个方法中，我们就可以定义如何处理这些数据。

MyDataModelListener还包含doAfterAllAnalysed方法，这个方法在所有数据都读取完毕后被调用。这里可以执行一些清理工作，或处理剩余的数据。

接下来，我们来实现这个 ReadListener：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import com.alibaba.excel.context.AnalysisContext;
    import com.alibaba.excel.read.listener.ReadListener;
    import org.springframework.transaction.annotation.Transactional;
    import java.util.ArrayList;
    import java.util.List;

    // 自定义的ReadListener，用于处理从Excel读取的数据
    public class MyDataModelListener implements ReadListener<MyDataModel> {
        // 设置批量处理的数据大小
        private static final int BATCH_SIZE = 1000;
        // 用于暂存读取的数据，直到达到批量大小
        private List<MyDataModel> batch = new ArrayList<>();

        
        private MyDataService myDataService;

        // 构造函数，注入MyBatis的Mapper
        public MyDataModelListener(MyDataService myDataService) {
            this.myDataService = myDataService;
        }

        // 每读取一行数据都会调用此方法
        @Override
        public void invoke(MyDataModel data, AnalysisContext context) {
            //检查数据的合法性及有效性
            if (validateData(data)) {
                //有效数据添加到list中
                batch.add(data);
            } else {
                // 处理无效数据，例如记录日志或跳过
            }
            
            // 当达到批量大小时，处理这批数据
            if (batch.size() >= BATCH_SIZE) {
                processBatch();
            }
        }

        
        private boolean validateData(MyDataModel data) {
            // 调用mapper方法来检查数据库中是否已存在该数据
            int count = myDataService.countByColumn1(data.getColumn1());
            // 如果count为0，表示数据不存在，返回true；否则返回false
            if(count == 0){
            	return true;
            }
            
            // 在这里实现数据验证逻辑
            return false;
        }


        // 所有数据读取完成后调用此方法
        @Override
        public void doAfterAllAnalysed(AnalysisContext context) {
            // 如果还有未处理的数据，进行处理
            if (!batch.isEmpty()) {
                processBatch();
            }
        }

        // 处理一批数据的方法，重试次数超过 3，进行异常处理
        private void processBatch() {
            int retryCount = 0;
            // 重试逻辑
            while (retryCount < 3) {
                try {
                    // 尝试批量插入
                    myDataService.batchInsert(batch);
                    // 清空批量数据，以便下一次批量处理
                    batch.clear();
                    break;
                } catch (Exception e) {
                    // 重试计数增加
                    retryCount++;
                    // 如果重试3次都失败，记录错误日志
                    if (retryCount >= 3) {
                        logError(e, batch);
                    }
                }
            }
        }

       

        // 记录错误日志的方法
        private void logError(Exception e, List<MyDataModel> failedBatch) {
            // 在这里实现错误日志记录逻辑
            // 可以记录异常信息和导致失败的数据
        }
    }

    @Service
    public class MyDataService{
        // MyBatis的Mapper，用于数据库操作
        @Autowired
        private MyDataMapper myDataMapper;
        
     	// 使用Spring的事务管理进行批量插入
        @Transactional(rollbackFor = Exception.class)
        public void batchInsert(List<MyDataModel> batch) {
            // 使用MyBatis Mapper进行批量插入
            myDataMapper.batchInsert(batch);
        }

        public int countByColumn1(String column1){
            return myDataMapper.countByColumn1(column1);
        }
        
    }
              
通过自定义这个 MyDataModelListener，我们就可以在读取Excel文件的过程中处理数据。

每读取到一条数据之后会把他们放入一个 List，当List中积累到1000条之后，进行一次数据库的批量插入，插入时如果失败了则重试，最后还是失败就打印日志。

这里批量插入，用到了 MyBatis 的批量插入，代码实现如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import org.apache.ibatis.annotations.Mapper;
    import java.util.List;

    @Mapper
    public interface MyDataMapper {
        void batchInsert(List<MyDataModel> dataList);

        int countByColumn1(String column1);
    }
              
mapper.xml 文件如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <insert id="batchInsert" parameterType="list">
        INSERT INTO test_table_name (column1, column2, ...)
        VALUES 
        <foreach collection="list" item="item" index="index" separator=",">
            (#{item.column1}, #{item.column2}, ...)
        </foreach>
    </insert>

    <select id="countByColumn1" resultType="int">
        SELECT COUNT(*) FROM your_table WHERE column1 = #{column1}
    </select>
              
基于 EasyExcel + 线程池解决 POI 文件导出溢出问题 {#easy-excel-poi}
===================================================

背景
---

在 CRM 后台管理系统中，需要导出 Excel ，但是在处理大数据量的 Excel 文件导出的时候，常用的 Apache POI 库可能因为内存占用太高导致内存的溢出，同时，数据处理的过程中耗时非常久，从而导致用户等待时间过长或者请求超时，为了解决这些问题，采用了 EasyExcel + 线程池的解决方案。

原因探究：为什么 POI 会导致内存溢出？ {#poi}
----------------------------

首先由两个方面的原因，第一个方面就是 Excel 实际上没有我们看的得那么小，其在加载的时候采用了压缩的策略，另外一个方面就是 POI 文件的处理会将整个 Excel 文档加载到内存中，从而导致处理大型文件的时候导致内存溢出。

### 为什么 Excel 没有我们看起来那么小 {#excel}

我们常见的 Excel 文件其是是一个压缩文件，它是将若干个 XML 格式的纯文本压缩在一起，Excel 就是读取这些压缩文件的信息，最后展现为一个表格文件。

所以，如果我们把xlsx文件的后缀更改为.zip或.rar，再进行解压缩，就能提取出构成Excel的核心源码文件。解压后会发现解压后的文件中有3个文件夹和1个XML格式文件：

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/32973aadf6624df3ad6a712d906d5d25~tplv-73owjymdk6-watermark.image?policy=eyJ2bSI6MywidWlkIjoiMTI4MDE3MTc1OTQ0NTU3In0%3D&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1721052090&x-orig-sign=332C5hvnNupipNzxt3AHEFkAOt0%3D)

*  
_rels 文件夹：里面的数据是一些基础的配置信息，比如 workbook 文件的位置等信息  
*  
docProps 文件夹：主要存储的是 sheet 的元信息，如果需要创建和编辑 sheet，一般修改的都是这个文件的内容  
*  
  xl 文件夹：这个是最重要的一个文件夹，里面存放了 sheet 中的数据、行和列的格式、单元格格式、sheet 的配置信息等

综上所示，我们可以发现我们处理的 excel 文件实际上是一个经过高度压缩的文件格式，背后有许多的文件，所以看到的一个文件可能不止 2M，实际上会很大。

也就是说，我们在使用POI 处理 Excel 文件的时候，实际的大小可能远远大于我们所看见的大小，也就解释了为什么处理的文件只有 100 MB，最后实际内存占用却达到了 1GB 甚至更大。

### POI 文件溢出的原理 {#poi}

这里我们拿一个示例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import org.apache.poi.ss.usermodel.*;
    import org.apache.poi.xssf.usermodel.XSSFWorkbook;

    import java.io.File;
    import java.io.FileInputStream;
    import java.io.IOException;

    public class ExcelReadTest {

        public static void main(String[] args) {
            // 指定要读取的文件路径
            String filename = "example.xlsx";

            try (FileInputStream fileInputStream = new FileInputStream(new File(filename))) {
                // 创建工作簿对象
                Workbook workbook = new XSSFWorkbook(fileInputStream);

                // 获取第一个工作表
                Sheet sheet = workbook.getSheetAt(0);

                // 遍历所有行
                for (Row row : sheet) {
                    // 遍历所有单元格
                    for (Cell cell : row) {
                        Thread.sleep(100);
                        // 根据不同数据类型处理数据
                        switch (cell.getCellType()) {
                            case STRING:
                                System.out.print(cell.getStringCellValue() + "\t");
                                break;
                            case NUMERIC:
                                if (DateUtil.isCellDateFormatted(cell)) {
                                    System.out.print(cell.getDateCellValue() + "\t");
                                } else {
                                    System.out.print(cell.getNumericCellValue() + "\t");
                                }
                                break;
                            case BOOLEAN:
                                System.out.print(cell.getBooleanCellValue() + "\t");
                                break;
                            case FORMULA:
                                System.out.print(cell.getCellFormula() + "\t");
                                break;
                            default:
                                System.out.print(" ");
                        }
                    }
                    System.out.println();
                }
            } catch (IOException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }

    }
              
这里面使用了一个关键的 XSSFWorkbook 类：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public XSSFWorkbook(InputStream is) throws IOException {
        this(PackageHelper.open(is));
    }

    public static OPCPackage open(InputStream is) throws IOException {
        try {
            return OPCPackage.open(is);
        } catch (InvalidFormatException e){
            throw new POIXMLException(e);
        }
    }
              
最后会调用 OPCPackage.open 方法  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public static OPCPackage open(InputStream in) throws InvalidFormatException,
            IOException {
        OPCPackage pack = new ZipPackage(in, PackageAccess.READ_WRITE);
        try {
            if (pack.partList == null) {
                pack.getParts();
            }
        } catch (InvalidFormatException | RuntimeException e) {
            IOUtils.closeQuietly(pack);
            throw e;
        }
        return pack;
    }
              
然后我们可以看一下这个方法相关的注释：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    
    /**
     * Open a package.
     *
     * Note - uses quite a bit more memory than {@link #open(String)}, which
     * doesn't need to hold the whole zip file in memory, and can take advantage
     * of native methods
     *
     * @param in
     *            The InputStream to read the package from
     * @return A PackageBase object
     *
     * @throws InvalidFormatException
     * 				Throws if the specified file exist and is not valid.
     * @throws IOException If reading the stream fails
     */
              
其中说了，这个方法会将整个压缩文件都加载到内存中，显而易见，就是将 Excel 的文档格式全都加载到内存中了，所以在处理小型文件的时候可能没什么影响，一旦处理大型文件，内存肯定是会溢出的。

技术选型
----

Excel 的导出有很多种方案，包括 POI、EasyExcel 以及 Hutool 等，市面上比较常用的解决方案主要是 POI 和 EasyExcel，然后在处理大文件这个方面，EasyExcel 更加适合一点，其内存占用更少，而且在处理大文件方面，EasyExcel 更加适合一些。

在文件导出的过程中，可以使用异步的方式去进行，用户不用一直等待，在异步文件生成之后，把文件上床到云存储中，再通知用户去下载就可以了。

然后云存储这块我们可以自由选择，这里选择使用阿里云 OSS，线程池异步处理采用 @Async

用户通知可以选择邮件通知的方式，这里使用 Spring Main 邮件发送。

具体实现
----

### 入口层

入口是一个Controller，主要接收用户的文件导出请求。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @RestController
    @RequestMapping("/export")
    public class DataExportController {

        @Autowired
        private ExcelExportService exportService;

        @GetMapping("/data")
        public ResponseEntity<String> exportData() {
            List<DataModel> data = fetchData();
            String fileUrl = exportService.exportDataAsync(data);

            return ResponseEntity.ok("导出任务开始，文件生成后会通知您下载链接");
        }

        private List<DataModel> fetchData() {
            // 获取需要导出的数据
        }
    }
              
然后在文件获取的时候，可能需要一些具体的数据获取，这个流程这里选就没有细说，可以结合业务场景进行实际的评估。

### 导出业务实现

这里主要有三个方面的内容：

*  
使用 EasyExcel 生成文件  
*  
OSS 上传生成后的文件  
*  
  Spring Mail 给用户发通知

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Service
    public class ExcelExportService {

        @Async("exportExecutor")
        public String exportDataAsync(List<DataModel> data) {
            // 生成 Excel 文件并获取 InputStream
            InputStream fileContent = generateExcelFile(data);
            String fileName = "data_" + System.currentTimeMillis() + ".xlsx";
            
            // 上传到 OSS
            String fileUrl = ossService.uploadFile(fileName, fileContent);
            // 发送邮件
            emailService.sendEmail(data.getUserEmail(), "文件导出通知", "您的文件已导出，下载链接: " + fileUrl);
            return fileUrl;
        }

       private InputStream generateExcelFile(List<DataModel> data) {
            ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
            try {
                ExcelWriterBuilder writerBuilder = EasyExcel.write(outputStream, DataModel.class);
                writerBuilder.sheet("Data").doWrite(data);
            } catch (Exception e) {
                // 处理异常
            }
            return new ByteArrayInputStream(outputStream.toByteArray());
        }

        // DataModel 类定义
        public static class DataModel {
            //省略参数及setter/getter
        }
    }
              
这里还用到了一个线程池的技术：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Configuration
    @EnableAsync
    public class AsyncExecutorConfig {
        @Bean("exportExecutor")
        public Executor exportExecutor() {

            ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
                    .setNameFormat("registerSuccessExecutor-%d").build();

            ExecutorService executorService = new ThreadPoolExecutor(10, 20,
                    0L, TimeUnit.MILLISECONDS,
                    new LinkedBlockingQueue<Runnable>(1024), namedThreadFactory, new ThreadPoolExecutor.AbortPolicy());

            return executorService;
        }

    }
              
OSS上传服务部分代码实现如下，依赖阿里云OSS的API进行文件上传：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import com.aliyun.oss.OSS;
    import com.aliyun.oss.OSSClientBuilder;
    import com.aliyun.oss.model.PutObjectRequest;

    import java.io.InputStream;
    import java.net.URL;
    import java.util.Date;

    public class OssService {

        private String endpoint = "<OSS_ENDPOINT>";
        private String accessKeyId = "<ACCESS_KEY_ID>";
        private String accessKeySecret = "<ACCESS_KEY_SECRET>";
        private String bucketName = "<BUCKET_NAME>";

        public String uploadFile(String fileName, InputStream fileContent) {
            OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);
            try {
                ossClient.putObject(new PutObjectRequest(bucketName, fileName, fileContent));
                // 设置URL过期时间为1小时
                Date expiration = new Date(System.currentTimeMillis() + 3600 * 1000);
                URL url = ossClient.generatePresignedUrl(bucketName, fileName, expiration);
                return url.toString();
            } finally {
                if (ossClient != null) {
                    ossClient.shutdown();
                }
            }
        }
    }
              
邮件发送代码：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.mail.javamail.JavaMailSender;
    import org.springframework.mail.SimpleMailMessage;
    import org.springframework.stereotype.Service;

    @Service
    public class EmailNotificationService {

        @Autowired
        private JavaMailSender mailSender;

        public void sendEmail(String toAddress, String subject, String body) {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom("noreply@example.com");
            message.setTo(toAddress);
            message.setSubject(subject);
            message.setText(body);

            mailSender.send(message);
        }
    }
              
### 配置文件参数

主要配置一下 Spring Mail 的配置信息：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    spring.mail.host=192.168.88.101
    spring.mail.port=8907
    spring.mail.username=user@gmail.com
    spring.mail.password=123456
    spring.mail.properties.mail.smtp.auth=true
    spring.mail.properties.mail.smtp.starttls.enable=true
              
基于 EasyExcel + 线程池 + 批量插入实现百万级数据导入 {#easy-excel}
================================================

背景
---

项目中有一个数据迁移，原来的数据存储在旧的系统，现在系统做了重构，需要迁移到新的系统中，老系统的数据被加工到Excel中了，需要基于Excel实现文件的导入，同时需要避免内存溢出以及性能太低的问题。

问题分析
----

* 1.  
  **内存溢出问题**

*  
  百万级别的数据量的 Excel 文件会非常大，如果全都加载到内存中，可能会导致内存溢出问题。

* 1.  
  **性能问题**

*  
  百万级别数据从 Excel 读取并且插入到数据中， 可能会很慢，所以需要考虑性能来问题

* 1.  
  **错误处理**

*  
  在文件的读取以及导入过程中，可能会出现各种各样的问题，我们要妥善解决好这些问题。

内存溢出问题
------

百万级别数据量，如果一次性读取到内存中，肯定是不现实的，那么最好的办法就是**基于流式读取的方式进行分批处理**。

在技术选型上面，我们选择使用 EasyExcel，特别是针对大数据量和复杂 Excel 文件的处理进行优化。在解析 Excel 的时候，EasyExcel 不会一次性将 Excel 一次性全部加载到内存中，而是从磁盘上面一行行进行读取，逐步进行解析。

性能问题
----

百万级别的数据量，如果采用单线程进行读取的话，性能会非常非常 low ！！！

所以，针对这个问题，我们就需要使用**多线程。**

使用多线程主要涉及到两个场景：

*  
多线程读取  
*  
  多线程实现数据插入

这里就涉及到一个生产者---消费者的模式，通过使用多个线程进行读取，然后多个线程实现插入，从而最大限度地提升整体的性能来。

数据的插入，除了借助多线程以外，还可以同时使用**数据库的批量插入的功能，** 从而提高插入的速度。

错误处理
----

在文件的读取和数据库写入的过程中，需要解决各种各样的问题，比如数据格式错误、数据不一致、重复数据等问题。

所以我们需要分两步进行一个处理：

* 1.  
对数据进行检查，在开始插入之前将数据的格式问题提前检查好  
* 2.  
  在插入过程中，需要对异常进行处理

对于异常处理，我们的处理方式有很多，可以进行日志记录，也可以选择进行事务回滚，这个主要根据实际情况进行选择，一般情况下是不建议进行回滚的，直接自动重试，如果重试之后还是不行，则记录日志，再记录日志然后重新插入即可。

并且在这个过程中，需要考虑一下数据重复的问题，需要在 Excel 中某几个字段设置成数据库唯一约束，然后遇到数据冲突的情况，可以先进行处理，处理的方式可以是覆盖、跳过以及报错，这个根据业务的实际情况来确定，一般可以使用跳过+打印日志实现。

技术选型
----

在大文件的读取方面，EasyExcel更合适，因为他不会像POI一样耗内存，可以大大的减少内存占用。因为他并不会一次性把整个Excel都加载到内存中，而是逐行读取的。

同时考虑使用多线程来读取，这里就需要用到线程池的技术，直接用ExecutorService就行了。

因为还涉及到数据的批量写入，需要依赖mybatis或者mybatis-plus。

整体方案
----

使用 EasyExcel 实现 Excel 文件的读取，因为他不会一次性将整个 Excel 加载到内存中，是采取逐行读取的方式，并且为了提高并发性能，我们可以将数据先分散到不同的 sheet 中，然后借助线程池，多线程同时读取不同的sheet，在读取的过程中，借助 EasyExcel 的 ReadListener 进行处理。

在处理的过程中，我们并不会每一条数据都操作一次数据库，这样对于数据库的压力太大了，一般我们会设置一个批次，如 2000 条，我们从 Excel 中读取的数据暂时存储到内存中，这里可以使用 List 实现，在读取 2000 条数据，就可以执行一次数据的批量插入，这里可以使用 MyBatis 的批量插入以及 MP 的批量插入功能实现。

MyBatis 实现批量通过插入：<https://juejin.cn/post/7016691244973686820>


然后在这个过程中，需要考虑一些并发的问题，所以我们这里会使用一些线程安全的队列，比如 **ConcurrentLinkedQueue**

然后在经过验证以后，读取 100 w 的 Excel 并且插入数据之后，总耗时在 100 秒左右，用时不超过 2 分钟。

具体实现
----

为了提高并发处理能力，我们将百万级别的数据放到同一个 Excel 的不同 sheet 中， 然后通过使用 EasyExcel 并发读取这些 sheet。

EasyExcel 提供了 ReadListener 接口，允许在读取每一批数据之后自定义进行处理，我们可以基于这个功能来实现文件的分批读取。

### 依赖添加

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependencies>
    <!-- EasyExcel -->
    <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>最新的版本号</version>
    </dependency>

    <!-- 数据库连接和线程池 -->
    <dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>最新版本号</version>
    </dependency>
    <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
    </dependency>
    </dependencies>
              
### 代码实现

然后实现并发读取多个 sheet 代码：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Service
    public class ExcelImporterService {

        @Autowired
        private MyDataService myDataService;

        public void doImport() {
            // Excel文件的路径
            String filePath = "users/linqi/workspace/excel/test.xlsx";

            // 需要读取的sheet数量
            int numberOfSheets = 20;

            // 创建一个固定大小的线程池，大小与sheet数量相同
            ExecutorService executor = Executors.newFixedThreadPool(numberOfSheets);

            // 遍历所有sheets
            for (int sheetNo = 0; sheetNo < numberOfSheets; sheetNo++) {
                // 在Java lambda表达式中使用的变量需要是final
                int finalSheetNo = sheetNo;

                // 向线程池提交一个任务
                executor.submit(() -> {
                    // 使用EasyExcel读取指定的sheet
                    EasyExcel.read(filePath, MyDataModel.class, new MyDataModelListener(myDataService))
                    .sheet(finalSheetNo) // 指定sheet号
                    .doRead(); // 开始读取操作
                });
            }

            // 启动线程池的关闭序列
            executor.shutdown();

            // 等待所有任务完成，或者在等待超时前被中断
            try {
                executor.awaitTermination(Long.MAX_VALUE, TimeUnit.NANOSECONDS);
            } catch (InterruptedException e) {
                // 如果等待过程中线程被中断，打印异常信息
                e.printStackTrace();
            }
        }
    }
              
这段代码通过创建一个固定大小的线程池来并发读取一个包含多个sheets的Excel文件。每个sheet的读取作为一个单独的任务提交给线程池。

我们在代码中用了一个MyDataModelListener，这个类是ReadListener的一个实现类。当EasyExcel读取每一行数据时，它会自动调用我们传入的这个ReadListener实例的invoke方法。在这个方法中，我们就可以定义如何处理这些数据。

MyDataModelListener还包含doAfterAllAnalysed方法，这个方法在所有数据都读取完毕后被调用。这里可以执行一些清理工作，或处理剩余的数据。

接下来，我们来实现这个 ReadListener：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import com.alibaba.excel.context.AnalysisContext;
    import com.alibaba.excel.read.listener.ReadListener;
    import org.springframework.transaction.annotation.Transactional;
    import java.util.ArrayList;
    import java.util.List;

    // 自定义的ReadListener，用于处理从Excel读取的数据
    public class MyDataModelListener implements ReadListener<MyDataModel> {
        // 设置批量处理的数据大小
        private static final int BATCH_SIZE = 1000;
        // 用于暂存读取的数据，直到达到批量大小
        private List<MyDataModel> batch = new ArrayList<>();

        
        private MyDataService myDataService;

        // 构造函数，注入MyBatis的Mapper
        public MyDataModelListener(MyDataService myDataService) {
            this.myDataService = myDataService;
        }

        // 每读取一行数据都会调用此方法
        @Override
        public void invoke(MyDataModel data, AnalysisContext context) {
            //检查数据的合法性及有效性
            if (validateData(data)) {
                //有效数据添加到list中
                batch.add(data);
            } else {
                // 处理无效数据，例如记录日志或跳过
            }
            
            // 当达到批量大小时，处理这批数据
            if (batch.size() >= BATCH_SIZE) {
                processBatch();
            }
        }

        
        private boolean validateData(MyDataModel data) {
            // 调用mapper方法来检查数据库中是否已存在该数据
            int count = myDataService.countByColumn1(data.getColumn1());
            // 如果count为0，表示数据不存在，返回true；否则返回false
            if(count == 0){
            	return true;
            }
            
            // 在这里实现数据验证逻辑
            return false;
        }


        // 所有数据读取完成后调用此方法
        @Override
        public void doAfterAllAnalysed(AnalysisContext context) {
            // 如果还有未处理的数据，进行处理
            if (!batch.isEmpty()) {
                processBatch();
            }
        }

        // 处理一批数据的方法，重试次数超过 3，进行异常处理
        private void processBatch() {
            int retryCount = 0;
            // 重试逻辑
            while (retryCount < 3) {
                try {
                    // 尝试批量插入
                    myDataService.batchInsert(batch);
                    // 清空批量数据，以便下一次批量处理
                    batch.clear();
                    break;
                } catch (Exception e) {
                    // 重试计数增加
                    retryCount++;
                    // 如果重试3次都失败，记录错误日志
                    if (retryCount >= 3) {
                        logError(e, batch);
                    }
                }
            }
        }

       

        // 记录错误日志的方法
        private void logError(Exception e, List<MyDataModel> failedBatch) {
            // 在这里实现错误日志记录逻辑
            // 可以记录异常信息和导致失败的数据
        }
    }

    @Service
    public class MyDataService{
        // MyBatis的Mapper，用于数据库操作
        @Autowired
        private MyDataMapper myDataMapper;
        
     	// 使用Spring的事务管理进行批量插入
        @Transactional(rollbackFor = Exception.class)
        public void batchInsert(List<MyDataModel> batch) {
            // 使用MyBatis Mapper进行批量插入
            myDataMapper.batchInsert(batch);
        }

        public int countByColumn1(String column1){
            return myDataMapper.countByColumn1(column1);
        }
        
    }
              
通过自定义这个 MyDataModelListener，我们就可以在读取Excel文件的过程中处理数据。

每读取到一条数据之后会把他们放入一个 List，当List中积累到1000条之后，进行一次数据库的批量插入，插入时如果失败了则重试，最后还是失败就打印日志。

这里批量插入，用到了 MyBatis 的批量插入，代码实现如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import org.apache.ibatis.annotations.Mapper;
    import java.util.List;

    @Mapper
    public interface MyDataMapper {
        void batchInsert(List<MyDataModel> dataList);

        int countByColumn1(String column1);
    }
              
mapper.xml 文件如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <insert id="batchInsert" parameterType="list">
        INSERT INTO test_table_name (column1, column2, ...)
        VALUES 
        <foreach collection="list" item="item" index="index" separator=",">
            (#{item.column1}, #{item.column2}, ...)
        </foreach>
    </insert>

    <select id="countByColumn1" resultType="int">
        SELECT COUNT(*) FROM your_table WHERE column1 = #{column1}
    </select>
              
基于 EasyExcel + 线程池解决 POI 文件导出溢出问题 {#easy-excel-poi}
===================================================

背景
---

在 CRM 后台管理系统中，需要导出 Excel ，但是在处理大数据量的 Excel 文件导出的时候，常用的 Apache POI 库可能因为内存占用太高导致内存的溢出，同时，数据处理的过程中耗时非常久，从而导致用户等待时间过长或者请求超时，为了解决这些问题，采用了 EasyExcel + 线程池的解决方案。

原因探究：为什么 POI 会导致内存溢出？ {#poi}
----------------------------

首先由两个方面的原因，第一个方面就是 Excel 实际上没有我们看的得那么小，其在加载的时候采用了压缩的策略，另外一个方面就是 POI 文件的处理会将整个 Excel 文档加载到内存中，从而导致处理大型文件的时候导致内存溢出。

### 为什么 Excel 没有我们看起来那么小 {#excel}

我们常见的 Excel 文件其是是一个压缩文件，它是将若干个 XML 格式的纯文本压缩在一起，Excel 就是读取这些压缩文件的信息，最后展现为一个表格文件。

所以，如果我们把xlsx文件的后缀更改为.zip或.rar，再进行解压缩，就能提取出构成Excel的核心源码文件。解压后会发现解压后的文件中有3个文件夹和1个XML格式文件：

![](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/32973aadf6624df3ad6a712d906d5d25~tplv-73owjymdk6-watermark.image?policy=eyJ2bSI6MywidWlkIjoiMTI4MDE3MTc1OTQ0NTU3In0%3D&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1721052090&x-orig-sign=332C5hvnNupipNzxt3AHEFkAOt0%3D)

*  
_rels 文件夹：里面的数据是一些基础的配置信息，比如 workbook 文件的位置等信息  
*  
docProps 文件夹：主要存储的是 sheet 的元信息，如果需要创建和编辑 sheet，一般修改的都是这个文件的内容  
*  
  xl 文件夹：这个是最重要的一个文件夹，里面存放了 sheet 中的数据、行和列的格式、单元格格式、sheet 的配置信息等

综上所示，我们可以发现我们处理的 excel 文件实际上是一个经过高度压缩的文件格式，背后有许多的文件，所以看到的一个文件可能不止 2M，实际上会很大。

也就是说，我们在使用POI 处理 Excel 文件的时候，实际的大小可能远远大于我们所看见的大小，也就解释了为什么处理的文件只有 100 MB，最后实际内存占用却达到了 1GB 甚至更大。

### POI 文件溢出的原理 {#poi}

这里我们拿一个示例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import org.apache.poi.ss.usermodel.*;
    import org.apache.poi.xssf.usermodel.XSSFWorkbook;

    import java.io.File;
    import java.io.FileInputStream;
    import java.io.IOException;

    public class ExcelReadTest {

        public static void main(String[] args) {
            // 指定要读取的文件路径
            String filename = "example.xlsx";

            try (FileInputStream fileInputStream = new FileInputStream(new File(filename))) {
                // 创建工作簿对象
                Workbook workbook = new XSSFWorkbook(fileInputStream);

                // 获取第一个工作表
                Sheet sheet = workbook.getSheetAt(0);

                // 遍历所有行
                for (Row row : sheet) {
                    // 遍历所有单元格
                    for (Cell cell : row) {
                        Thread.sleep(100);
                        // 根据不同数据类型处理数据
                        switch (cell.getCellType()) {
                            case STRING:
                                System.out.print(cell.getStringCellValue() + "\t");
                                break;
                            case NUMERIC:
                                if (DateUtil.isCellDateFormatted(cell)) {
                                    System.out.print(cell.getDateCellValue() + "\t");
                                } else {
                                    System.out.print(cell.getNumericCellValue() + "\t");
                                }
                                break;
                            case BOOLEAN:
                                System.out.print(cell.getBooleanCellValue() + "\t");
                                break;
                            case FORMULA:
                                System.out.print(cell.getCellFormula() + "\t");
                                break;
                            default:
                                System.out.print(" ");
                        }
                    }
                    System.out.println();
                }
            } catch (IOException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }

    }
              
这里面使用了一个关键的 XSSFWorkbook 类：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public XSSFWorkbook(InputStream is) throws IOException {
        this(PackageHelper.open(is));
    }

    public static OPCPackage open(InputStream is) throws IOException {
        try {
            return OPCPackage.open(is);
        } catch (InvalidFormatException e){
            throw new POIXMLException(e);
        }
    }
              
最后会调用 OPCPackage.open 方法  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public static OPCPackage open(InputStream in) throws InvalidFormatException,
            IOException {
        OPCPackage pack = new ZipPackage(in, PackageAccess.READ_WRITE);
        try {
            if (pack.partList == null) {
                pack.getParts();
            }
        } catch (InvalidFormatException | RuntimeException e) {
            IOUtils.closeQuietly(pack);
            throw e;
        }
        return pack;
    }
              
然后我们可以看一下这个方法相关的注释：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    
    /**
     * Open a package.
     *
     * Note - uses quite a bit more memory than {@link #open(String)}, which
     * doesn't need to hold the whole zip file in memory, and can take advantage
     * of native methods
     *
     * @param in
     *            The InputStream to read the package from
     * @return A PackageBase object
     *
     * @throws InvalidFormatException
     * 				Throws if the specified file exist and is not valid.
     * @throws IOException If reading the stream fails
     */
              
其中说了，这个方法会将整个压缩文件都加载到内存中，显而易见，就是将 Excel 的文档格式全都加载到内存中了，所以在处理小型文件的时候可能没什么影响，一旦处理大型文件，内存肯定是会溢出的。

技术选型
----

Excel 的导出有很多种方案，包括 POI、EasyExcel 以及 Hutool 等，市面上比较常用的解决方案主要是 POI 和 EasyExcel，然后在处理大文件这个方面，EasyExcel 更加适合一点，其内存占用更少，而且在处理大文件方面，EasyExcel 更加适合一些。

在文件导出的过程中，可以使用异步的方式去进行，用户不用一直等待，在异步文件生成之后，把文件上床到云存储中，再通知用户去下载就可以了。

然后云存储这块我们可以自由选择，这里选择使用阿里云 OSS，线程池异步处理采用 @Async

用户通知可以选择邮件通知的方式，这里使用 Spring Main 邮件发送。

具体实现
----

### 入口层

入口是一个Controller，主要接收用户的文件导出请求。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @RestController
    @RequestMapping("/export")
    public class DataExportController {

        @Autowired
        private ExcelExportService exportService;

        @GetMapping("/data")
        public ResponseEntity<String> exportData() {
            List<DataModel> data = fetchData();
            String fileUrl = exportService.exportDataAsync(data);

            return ResponseEntity.ok("导出任务开始，文件生成后会通知您下载链接");
        }

        private List<DataModel> fetchData() {
            // 获取需要导出的数据
        }
    }
              
然后在文件获取的时候，可能需要一些具体的数据获取，这个流程这里选就没有细说，可以结合业务场景进行实际的评估。

### 导出业务实现

这里主要有三个方面的内容：

*  
使用 EasyExcel 生成文件  
*  
OSS 上传生成后的文件  
*  
  Spring Mail 给用户发通知

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Service
    public class ExcelExportService {

        @Async("exportExecutor")
        public String exportDataAsync(List<DataModel> data) {
            // 生成 Excel 文件并获取 InputStream
            InputStream fileContent = generateExcelFile(data);
            String fileName = "data_" + System.currentTimeMillis() + ".xlsx";
            
            // 上传到 OSS
            String fileUrl = ossService.uploadFile(fileName, fileContent);
            // 发送邮件
            emailService.sendEmail(data.getUserEmail(), "文件导出通知", "您的文件已导出，下载链接: " + fileUrl);
            return fileUrl;
        }

       private InputStream generateExcelFile(List<DataModel> data) {
            ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
            try {
                ExcelWriterBuilder writerBuilder = EasyExcel.write(outputStream, DataModel.class);
                writerBuilder.sheet("Data").doWrite(data);
            } catch (Exception e) {
                // 处理异常
            }
            return new ByteArrayInputStream(outputStream.toByteArray());
        }

        // DataModel 类定义
        public static class DataModel {
            //省略参数及setter/getter
        }
    }
              
这里还用到了一个线程池的技术：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Configuration
    @EnableAsync
    public class AsyncExecutorConfig {
        @Bean("exportExecutor")
        public Executor exportExecutor() {

            ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
                    .setNameFormat("registerSuccessExecutor-%d").build();

            ExecutorService executorService = new ThreadPoolExecutor(10, 20,
                    0L, TimeUnit.MILLISECONDS,
                    new LinkedBlockingQueue<Runnable>(1024), namedThreadFactory, new ThreadPoolExecutor.AbortPolicy());

            return executorService;
        }

    }
              
OSS上传服务部分代码实现如下，依赖阿里云OSS的API进行文件上传：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import com.aliyun.oss.OSS;
    import com.aliyun.oss.OSSClientBuilder;
    import com.aliyun.oss.model.PutObjectRequest;

    import java.io.InputStream;
    import java.net.URL;
    import java.util.Date;

    public class OssService {

        private String endpoint = "<OSS_ENDPOINT>";
        private String accessKeyId = "<ACCESS_KEY_ID>";
        private String accessKeySecret = "<ACCESS_KEY_SECRET>";
        private String bucketName = "<BUCKET_NAME>";

        public String uploadFile(String fileName, InputStream fileContent) {
            OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);
            try {
                ossClient.putObject(new PutObjectRequest(bucketName, fileName, fileContent));
                // 设置URL过期时间为1小时
                Date expiration = new Date(System.currentTimeMillis() + 3600 * 1000);
                URL url = ossClient.generatePresignedUrl(bucketName, fileName, expiration);
                return url.toString();
            } finally {
                if (ossClient != null) {
                    ossClient.shutdown();
                }
            }
        }
    }
              
邮件发送代码：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.mail.javamail.JavaMailSender;
    import org.springframework.mail.SimpleMailMessage;
    import org.springframework.stereotype.Service;

    @Service
    public class EmailNotificationService {

        @Autowired
        private JavaMailSender mailSender;

        public void sendEmail(String toAddress, String subject, String body) {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom("noreply@example.com");
            message.setTo(toAddress);
            message.setSubject(subject);
            message.setText(body);

            mailSender.send(message);
        }
    }
              
### 配置文件参数

主要配置一下 Spring Mail 的配置信息：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    spring.mail.host=192.168.88.101
    spring.mail.port=8907
    spring.mail.username=user@gmail.com
    spring.mail.password=123456
    spring.mail.properties.mail.smtp.auth=true
    spring.mail.properties.mail.smtp.starttls.enable=true
              
![](/assets_dweb/logo@1x.png)  
知识星球  
扫码加入星球  
查看更多优质内容  
https://wx.zsxq.com/mweb/views/joingroup/join_group.html?group_id=48411118851818  
