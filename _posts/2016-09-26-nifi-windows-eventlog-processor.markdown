---
layout: post
title:  "Parsing evtx files with Apache NiFi"
date:   2016-09-26 12:38:00
categories: nifi windows eventlog
---
Windows Event Logs are stored and exported in the [evtx format][1].  This makes them difficult to work with in a programmatic fashion, especially outside the Windows ecosystem.  In order to facilitate the processing of Windows event data in Apache NiFi, a processor capable of parsing evtx files has been created.

It works using a parser [ported from Python][2] into pure Java and modified to read the file one chunk (64k) at a time to allow for processing large files without pulling the entire input FlowFile into memory.

The Processor expects incoming FlowFiles with evtx files as their contents.  It only requires the user to specify a desired granularity of output.  This allows the user to decide how the evtx file is broken up into individual FlowFiles with the event records translated to XML.

ParseEvtx has 4 output relationships.  It will output the unmodified FlowFile to the original relationship.  Any malformed chunks encountered will be sent to the bad chunk relationship.  Regardless of the configuration, the output and failure relationship FlowFiles will consist of a wrapping <Events/> xml tag with individual events inside.  The grouping of events within the output and failure relationships depends on the user-configured granularity property.

At File granularity, the entire evtx file will be put into a single output FlowFile.  This also means that if the file is corrupt or malformed in any way, the best-effort xml from parsing it will be sent to the failure relationship.

At Chunk granularity, each chunk (64k) of the evtx file will have its own FlowFile.  For any chunk that is detectably malformed via a bad checksum, there will not be XML failure output.  If a chunk isn't determined to be unparseable until the attempt is underway, the XML up until the error is discovered will be transferred via the failure relationship.

At Record granularity, every event will generate an output FlowFile.  The failure relationship isn't really relevant at this granularity because everything from the chunk up until the error has already been put into other FlowFiles.

In order to demonstrate this functionality, there is a specially crafted evtx file in the NiFi test resources designed to show the different possible paths.

First, download the [Apache NiFi archive.][nifidownload]

Then, download the [ParseEvtxSample flow.][template]

Use the following commands to prepare a demo workspace:

{% highlight shell %}
mkdir parse-evtx-demo/
cd parse-evtx-demo/
unzip ~/Downloads/nifi-1.0.0-bin.zip
nifi-1.0.0/bin/nifi.sh start
wget -P sample-data https://github.com/apache/nifi/raw/rel/nifi-1.0.0/nifi-nar-bundles/nifi-evtx-bundle/nifi-evtx-processors/src/test/resources/application-logs.evtx
{% endhighlight %}

Open the NiFi UI and upload the [template][template] and instantiate it on the canvas.

![Success]({{ site.baseurl }}/images/2016-09-26-parse-evtx-example.png)

Start the flow, every 10 seconds you should see output to each relationship.

In an output folder in parse-evtx-demo, you can see the contents of each different queue's FlowFile(s).

In the normal well formed file case, you should only see output to the success and original relationships; the other two are to make it possible to handle malformed or incorrectly parsed files. The sample evtx was doctored to exercise all of the processor's relationships.

You can change the granularity on the ParseEvtx processor and see how that impacts the different cases.

[1]:https://github.com/williballenthin/python-evtx/blob/master/documentation/p65-schuster.pdf
[2]:https://github.com/williballenthin/python-evtx
[nifidownload]:https://nifi.apache.org/download.html
[template]: /attachments/templates/ParseEvtxSample.xml
