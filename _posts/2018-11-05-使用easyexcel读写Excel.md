---
title: "使用easyexcel读写Excel"

url: "https://wsk1103.github.io/"

tags:
  - Java
  - 学习笔记
---


## JAVA解析Excel工具[easyexcel](https://github.com/alibaba/easyexcel)

Java解析、生成Excel比较有名的框架有Apache poi、jxl。但他们都存在一个严重的问题就是非常的耗内存，poi有一套SAX模式的API可以一定程度的解决一些内存溢出的问题，但POI还是有一些缺陷，比如07版Excel解压缩以及解压后存储都是在内存中完成的，内存消耗依然很大。  
easyexcel重写了poi对07版Excel的解析，能够原本一个3M的excel用POI sax依然需要100M左右内存降低到KB级别，并且再大的excel不会出现内存溢出，03版依赖POI的sax模式。在上层做了模型转换的封装，让使用者更加简单方便

### 环境 Java 1.7 +
### maven 3.0.5 +

### 1. 准备pom.xml

```
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>{latestVersion}</version>
</dependency>
```
目前最新版本是1.1.1（2018-11-5）  
**注意：** 该版本下使用的POI版本为**3.17**，所以当项目中的POI版本不为3.17时（有可能项目之前已经引入POI，easyexcel默认自带版本为3.17），可以考虑升级或者参考文末方法


### 2. 创建实体

假设Excel中列表为  
![image](https://raw.githubusercontent.com/wsk1103/images/master/easyexcel/1.png)


先创建相应的实体User.java  

```
@Data
public class User extends BaseRowModel {
    @ExcelProperty(value = "姓名", index = 0)
    private String name;

    @ExcelProperty(value = "昵称", index = 1)
    private String nickName;

    @ExcelProperty(value = "密码", index = 2)
    private String password;

    @ExcelProperty(value = "生日", index = 3, format = "yyyy/MM/dd")
    private Date birthday;
}
```
注意，该实体必须继承 **BaseRowModel**



### 3. 编写监听类，该类用于返回读取到的对象

```
/**
 * @author WuShukai
 * @version V1.0
 * @description 处理Excel，将读取到数据保存为对象并输出
 * @date 2018/11/6  16:44
 */
public class ExcelListener<T extends BaseRowModel> extends AnalysisEventListener<T> {
    /**
     * 自定义用于暂时存储data。
     * 可以通过实例获取该值
     */
    private final List<T> data = new ArrayList<>();

    @Override
    public void invoke(T object, AnalysisContext context) {
        //数据存储
        data.add(object);
    }

    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {

    }

    public List<T> getData() {
        return data;
    }

}
```


### 4. 编写工具类


```
    /**
    * 从Excel中读取文件，读取的文件是一个DTO类，该类必须继承BaseRowModel
    * 具体实例参考 ： MemberMarketDto.java
    * 参考：https://github.com/alibaba/easyexcel
    * 字符流必须支持标记，FileInputStream 不支持标记，可以使用BufferedInputStream 代替
    * BufferedInputStream bis = new BufferedInputStream(new FileInputStream(...));
    *
    * @param inputStream 文件输入流
    * @param clazz       继承该类必须继承BaseRowModel的类
    * @return 读取完成的list
    */
    public static <T extends BaseRowModel> List<T> readExcel(final InputStream inputStream, final Class<? extends BaseRowModel> clazz) {
        if (null == inputStream) {
            throw new NullPointerException("the inputStream is null!");
        }
        AnalysisEventListener listener = new ExcelListener();
        //读取xls 和 xlxs格式
        //如果POI版本为3.17，可以如下声明
        ExcelReader reader = new ExcelReader(inputStream, null, listener);
        //判断格式，针对POI版本低于3.17
        //ExcelTypeEnum excelTypeEnum = valueOf(inputStream);
        //ExcelReader reader = new ExcelReader(inputStream, excelTypeEnum, null, listener);
        reader.read(new com.alibaba.excel.metadata.Sheet(1, 1, clazz));

        return ((ExcelListener) listener).getData();
	}
	
    /**
    * 需要写入的Excel，有模型映射关系
    *
    * @param file  需要写入的Excel，格式为xlsx
    * @param list 写入Excel中的所有数据，继承于BaseRowModel
    */
    public static void writeExcel(final File file, List<? extends BaseRowModel> list) {
        OutputStream out = new FileOutputStream(file);
        try {
            ExcelWriter writer = new ExcelWriter(out, ExcelTypeEnum.XLSX);
            //写第一个sheet,  有模型映射关系
            Class t = list.get(0).getClass();
            com.alibaba.excel.metadata.Sheet sheet = new com.alibaba.excel.metadata.Sheet(1, 0, t);
            writer.write(list, sheet);
            writer.finish();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
	
	
    /**
    * 根据输入流，判断为xls还是xlsx，该方法原本存在于easyexcel 1.1.0 的ExcelTypeEnum中。
    * 如果POI版本为3.17以下，则FileMagic会报错，找不到该类，此时去到POI 3.17中将FileMagic抽取出来
    */
    public static ExcelTypeEnum valueOf(InputStream inputStream) {
        try {
            FileMagic fileMagic = FileMagic.valueOf(inputStream);
            if (FileMagic.OLE2.equals(fileMagic)) {
                return ExcelTypeEnum.XLS;
            }
            if (FileMagic.OOXML.equals(fileMagic)) {
                return ExcelTypeEnum.XLSX;
            }
            throw new IllegalArgumentException("excelTypeEnum can not null");

        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    
```




### 5. POI版本过低处理
注意：当POI版本低于easyexcel中内置的POI版本不一致的时候，只能使用被声明为过期的方法
 
```
ExcelReader reader = new ExcelReader(inputStream, ExcelTypeEnum.XLSX, null, new AnalysisEventListener<List<String>>() {...});
```

无法自动判断Excel为03还是07版本，此时可以将缺少POI 3.17 中的方法拷贝出来使用。


```

/**
 * @author WuShukai
 * @version V1.0
 * @description 判断格式，这个枚举存在于poi 3.17，但是目前版本是3.15，所以从3.17抽出来使用
 * @date 2018/11/6  16:46
 */
public enum FileMagic {
    /**
     * OLE2 / BIFF8+ stream used for Office 97 and higher documents
     */
    OLE2(HeaderBlockConstants._signature),
    /**
     * OOXML / ZIP stream
     */
    OOXML(org.apache.poi.poifs.common.POIFSConstants.OOXML_FILE_HEADER),
    /**
     * UNKNOWN magic
     */
    UNKNOWN(new byte[0]);

    final byte[][] magic;

    FileMagic(long magic) {
        this.magic = new byte[1][8];
        LittleEndian.putLong(this.magic[0], 0, magic);
    }

    FileMagic(byte[]... magic) {
        this.magic = magic;
    }

    public static FileMagic valueOf(byte[] magic) {
        for (FileMagic fm : values()) {
            int i = 0;
            boolean found = true;
            for (byte[] ma : fm.magic) {
                for (byte m : ma) {
                    byte d = magic[i++];
                    if (!(d == m || (m == 0x70 && (d == 0x10 || d == 0x20 || d == 0x40)))) {
                        found = false;
                        break;
                    }
                }
                if (found) {
                    return fm;
                }
            }
        }
        return UNKNOWN;
    }
    
    /**
     * @param inp An InputStream which supports either mark/reset
     */
    public static FileMagic valueOf(InputStream inp) throws IOException {
        if (!inp.markSupported()) {
            throw new IOException("getFileMagic() only operates on streams which support mark(int)");
        }

        // Grab the first 8 bytes
        byte[] data = IOUtils.peekFirst8Bytes(inp);

        return FileMagic.valueOf(data);
    }

}
```



### 6. InputStream无法标记错误,error for mark(in);
因为FileInputStream是无法被标记的，可以将FileInputStream替换成BufferedInputStream。

```
try(BufferedInputStream bis = new BufferedInputStream(new FileInputStream(file))) {
    do something...
}
```

