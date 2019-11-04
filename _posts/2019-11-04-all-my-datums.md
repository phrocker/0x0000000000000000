---
layout: post
title: All my datums
image: https://www.0x0000000000000000.com/img/large-895567_1920.jpg
---

<p>Apache Accumulo provides prefix encoding via something called "Relative Key Encoding"[1]. This amounts to removing identical portions of consecutive keys. For example if a key begins with '20191104_1' and the next key is '20191104_2' we don't need to store all bytes for the identical prefix. Instead we can store we only need to store the 2 and some metadata to identify the identical portions. If you're familiar with the concept of RFile, then you may be aware that the underlying design was the underpinnings to the aptly named storage file. </p>

<p>Data modeling is always quite difficult and is something I should never be trusted to do, but I wanted to spend a few minutes describing an experiment in which I ran tests of outputting data with similar key structures and those with completely randomized keys and values. In the case of relative key encoding the value is not touched. </p>

<p>Example one generating data with random UUIDs in the keys and values</p>
{% highlight java linenos %}
package org.poma.accumulo.nifi.processors;

import org.apache.accumulo.core.conf.AccumuloConfiguration;
import org.apache.accumulo.core.conf.DefaultConfiguration;
import org.apache.accumulo.core.data.Key;
import org.apache.accumulo.core.data.Value;
import org.apache.accumulo.core.file.FileSKVWriter;
import org.apache.accumulo.core.file.rfile.RFileOperations;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;
import java.util.SortedSet;
import java.util.TreeSet;
import java.util.UUID;
import java.util.concurrent.atomic.LongAdder;
import java.util.stream.IntStream;

public class WriterRandomData {

    public static void main(String [] args) throws IOException, URISyntaxException {
        AccumuloConfiguration config = new DefaultConfiguration();
        String filename = "file:///tmp/rfileon.rf";

        LoggerFactory.getLogger(WriterRandomData.class.getCanonicalName());
        Configuration hadoopConfig = new  Configuration();
        FileSystem fs = FileSystem.get(new URI(filename),hadoopConfig);
        FileSKVWriter writer = RFileOperations.getInstance().newWriterBuilder().forFile(filename,fs,hadoopConfig).withTableConfiguration(config).build();

        int keys = 1000000;

        SortedSet<Key> kv = new TreeSet<>();
        IntStream.range(0,keys).forEach( x -> {
                    Key key = new Key(UUID.randomUUID().toString(),UUID.randomUUID().toString(),UUID.randomUUID().toString());
                    kv.add(key);
        });

        writer.startDefaultLocalityGroup();
        LongAdder size = new LongAdder();
        kv.forEach( x ->{
            try {
                Value value = new Value(UUID.randomUUID().toString());
                size.add(x.getSize() + value.getSize());
                writer.append(x,value);
            } catch (IOException e) {
                e.printStackTrace();
            }
        });

        writer.close();
        System.out.println("Write " + size.longValue() + " bytes");

    }
}
{% endhighlight %}

<p>Example two where consecutive keys are created. Note that more keys are generated to replicate 'breaking the values out' amongst consecutive keys. </p>
{% highlight java linenos %}
package org.poma.accumulo.nifi.processors;

import org.apache.accumulo.core.conf.AccumuloConfiguration;
import org.apache.accumulo.core.conf.DefaultConfiguration;
import org.apache.accumulo.core.data.Key;
import org.apache.accumulo.core.data.Value;
import org.apache.accumulo.core.file.FileSKVWriter;
import org.apache.accumulo.core.file.rfile.RFileOperations;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;
import java.util.SortedSet;
import java.util.TreeSet;
import java.util.UUID;
import java.util.concurrent.atomic.LongAdder;
import java.util.stream.IntStream;

public class WriteAscendingData {

    public static void main(String [] args) throws IOException, URISyntaxException {
        AccumuloConfiguration config = new DefaultConfiguration();
        String filename = "file:///tmp/rfileonascending.rf";

        LoggerFactory.getLogger(WriteAscendingData.class.getCanonicalName());
        Configuration hadoopConfig = new  Configuration();
        FileSystem fs = FileSystem.get(new URI(filename),hadoopConfig);
        FileSKVWriter writer = RFileOperations.getInstance().newWriterBuilder().forFile(filename,fs,hadoopConfig).withTableConfiguration(config).build();

        int keys = 10000000; // also tried 30000000

        SortedSet<Key> kv = new TreeSet<>();
        IntStream.range(0,keys).forEach( x -> {
                    String leadingZero = org.apache.commons.lang.StringUtils.leftPad(Long.valueOf(x).toString(), 32, "0");
                    Key key = new Key("20191104","datasetABracadabralongvalueherebecausewehaveuuids",leadingZero);
                    kv.add(key);
        });

        writer.startDefaultLocalityGroup();
        LongAdder size = new LongAdder();
        kv.forEach( x ->{
            try {
                size.add(x.getSize());
                writer.append(x,new Value());
            } catch (IOException e) {
                e.printStackTrace();
            }
        });

        writer.close();
        System.out.println("Write " + size.longValue() + " bytes");

    }
}
{% endhighlight %}

<p>Results:</p>
<p>Across multiple runs with slightly varying key structures the results were nearly the same. The first code example writes out around 144 MB while the second example wrote out about 890 MB ( with the 30M rows being over 2GB ). The output files shows the relative value of RFiles and relative key encoding. The first example in which I generated 1M keys and values using UUID generated a 96 MB RFile everytime, while the second example generated anywhere from 1 MB to just over 1.5 MB when writing 30M keys and values. </p>
<p>I think it should be noted that the example is a bit contrived since they're not outputting the same exact data; however, this shows the value of
relative key encoding. The level of compression can be well into 100x and beyond depending on your data and what you are storing </p>
<p>[1] <a href="https://accumulo.apache.org/docs/2.x/getting-started/features#relative-encoding">https://accumulo.apache.org/docs/2.x/getting-started/features#relative-encoding</a></p>
