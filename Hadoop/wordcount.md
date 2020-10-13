## java word count

```java
import java.io.IOException;
//java的输入输bai出异常包，在控制台可以抛出输入输出错误
import java.util.StringTokenizer;
//是字符串分隔解析类型，构造一个用来解析str的StringTokenizer对象。
//java默认的分隔符是“空格”、“制表符（‘\t’）”、“换行符(‘\n’）”、“回车符（‘\r’）”。
import org.apache.hadoop.conf.Configuration;
// 获取文件系统
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

  /**
   * TokenizerMapper继承Mapper类
   * Mapper<KEYIN（输入key类型）, VALUEIN（输入value类型）, KEYOUT（输出key类型）, VALUEOUT（输出value类型）>
   */

  public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {

     // 因为若每个单词出现后，就置为 1，并将其作为一个<key,value>对，因此可以声明为常量，值为 1
    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    /**
     *重写map方法，读取初试划分的每一个键值对，即行偏移量和一行字符串，key为偏移量，value为该行字符串
     */
    public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
       /**
       * 因为每一行就是一个spilt，并会为之生成一个mapper，所以我们的参数，key就是偏移量，value就是一行字符串
       * value是一行的字符串，这里将其切割成多个单词,将每行的单词进行分割,按照"  \t\n\r\f"(空格、制表符、换行符、回车符、换页)进行分割
       */
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        //获取每个值并设置map输出的key值
        word.set(itr.nextToken());
        //one代表1，最开始每个单词都是1次，context直接将<word,1>写到本地磁盘上
        //write函数直接将两个参数封装成<key,value>
        context.write(word, one);
      }
    }
  }

  /**
   * IntSumReducer继承Reducer类
   * Reducer<KEYIN, VALUEIN, KEYOUT, VALUEOUT>:Map的输出类型，就是Reduce的输入类型
   */
  public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

    //输出结果，总次数
    private IntWritable result = new IntWritable();

    /**
     *重写reduce函数，key为单词，values是reducer从多个mapper中得到数据后进行排序并将相同key组
     *合成<key.list<V>>中的list<V>，也就是说明排序这些工作都是mapper和reducer自己去做的，
     *我们只需要专注与在map和reduce函数中处理排序处理后的结果
     */
    public void reduce(Text key, Iterable<IntWritable> values, Context context)
        throws IOException, InterruptedException {

      /**
       *因为在同一个spilt对应的mapper中，会将其进行combine，使得其中单词（key）不重复，然后将这些键值对按照
       *hash函数分配给对应的reducer，reducer进行排序，和组合成list，然后再调用的用户自定义的函数
       */
      int sum = 0;//累加器，累加每个单词出现的次数
      //遍历values
      for (IntWritable val : values) {
        sum += val.get();//累加
      }
      result.set(sum);//设置输出value
      context.write(key, result);//context输出reduce结果
    }
  }

  public static void main(String[] args) throws Exception {
    //获取配置信息
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");//创建一个job，设置名称
    job.setJarByClass(WordCount.class);//1、设置job运行的类
    //2、设置mapper类、Combiner类和Reducer类
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    //3、设置输出结果key和value的类
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    //4、为job设置输入路径
    FileInputFormat.addInputPath(job, new Path(args[0]));
    //5、为job设置输出路径
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    //6、结束程序
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}

```

## python word count

```python
# -*- coding:utf-8 -*-
from pyspark import SparkContext
from pyspark import SparkConf


def CreateSparkContext():
    sparkConf = SparkConf().setAppName("WordCounts").set(
        "spark.ui.showConsoleProgress", "false")
    sc = SparkContext(conf=sparkConf)
    print("master="+sc.master)  # 显示当前的运行模式
    SetLogger(sc)  # 设置不显示过多信息，函数在下面
    SetPath(sc)  # 设置文件读取路径，函数在下面
    return (sc)


def SetLogger(sc):  # 去除spark默认显示的杂乱信息
    logger = sc._jvm.org.apache.log4j
    logger.LogManager.getLogger("org").setLevel(logger.Level.ERROR)
    logger.LogManager.getLogger("akka").setLevel(logger.Level.ERROR)
    logger.LogManager.getRootLogger().setLevel(logger.Level.ERROR)


def SetPath(sc):  # 配置读取文件的路径（本地文件和hdfs文件）
    global Path
    if sc.master[0:5] == "local":
        # local模式读取本地文件
        Path = "file:/home/wfy/pythonworkspace/PythonProject/"
    else:  # 其他模式（yarn等）读取hdfs文件
        Path = "hdfs://localhost:9000/user/wfy/"


if __name__ == "__main__":
    print("开始执行 wordcount")
    sc = CreateSparkContext()
    print("开始读取文本文件...")
    textFile = sc.textFile(Path+"data/README.md")
    print("文本共有"+str(textFile.count())+"行")
    # map-reduce 运算
    countsRDD = textFile.flatMap(lambda line: line.split(' ')).map(
        lambda x: (x, 1)).reduceByKey(lambda x, y: x+y)
    print("文本统计共有"+str(countsRDD.count())+"项数据")
    # 保存结果到文件
    print("开始保存到文本文件...")
    try:
        countsRDD.saveAsTextFile(Path+"data/output")
    except Exception as e:
        print("输出目录已存在，请先删除原有目录")
    sc.stop()

```

