# Decoupled-pipelines-in-GStreamer
This is a brief post explaining the concept of decoupled pipelines in GStreamer


GStreamer is a multimedia framework, that enables media playback using a number of in-built elements/modules. GStreamer uses the concept
of pipelines to play the media. A pipeline is constructed of several elements. Elements are analagous to individual modules through which
the flow of data takes place. Each pipeline has a source, from where the data flow starts and a sink, where the data flow ends. An
example of a basic pipeline

file_src -> video_sink : `gst-launch filesrc location=abc.avi ! autovideosink `

video_src -> file_sink  : `gst-launch videotestsrc ! filesink location=xyz.avi `

A more complex example of pipeline would be to include live camera source, encoding/decoding, videorate and other elements:
`gst-launch v4l2src ! videorate rate = 2 ! x264enc ! autovideosink `

While working on a more complex pipelines, I found myself stuck in a  task, where the pipeline had to be modified dynamically based 
on user input. A new element had to inserted/deleted in between the pipeline based on user input. Doing this, resulted in the pipeline 
getting stalled because of the behaviour of the element being inserted/deleted.  This led to a series of nights, where I had to debug and
run through the logs generated by Gstreamer tools to understand the pipeline behaviour. Finally, I found out that the insertion of an element
resulted in a signal `EOS` being sent upstream that stalled the source element, and eventually the complete pipeline. 

So in cases, where a large number of elements are involved, or when one part of pipeline should not affect the other parts of pipelines, it 
makes sense to decouple te pipelines. This ensures the upstream and downstream pipeline works seamlessly even though one of them stalls.
For eg. in case the pipeline needs to read from a network source, and for some reason there is no data on the network port, a 
normal pipeline would stall. However a decoupled pipeline can still work with the last received data, avoiding a stalling behaviour. GStreamer
would run in threads for a single pipeline and using a decoupled pipeline, can lead to better handling with multiple threads running 
independently. Some example of decoupled pipeline elements are `intervideosink`, `intervideosrc`, `appsink` , `appsrc`, `fdsrc`, `fdsink`,
etc.

Following is a simple decoupled pipeline, that aims at explaining the concept of decoupling. Considering that we receive h264 encoded 
video packets on a network port using UDP, we receive them using element `udpsrc` and decode them and finally display them. The element 
`rotate` is added to rotate the video by any specified angle to understand the behaviour of decoupling.

Complete pipeline = pipeline source + pipeline sink

Pipeline source: ` udpsrc ! h264parse ! avdec_h264 ! intervideosink`

Pipeline sink :  ` intervideosrc ! rotate ! autovideosink `


In case now, the `udpsrc` , does not have data for next 10 seconds, the element `autovideosink` will still display the last 
received frame, and allow the element `rotate` to apply the rotation transformation as well. Note that in case of a normal pipeline,
and same scenario, the element `autovideosink` will display a black window and the complete pipeline will stall untill the data is
available again at  `udpsrc`.


The same concept can be applied to a pipeline, that consists of multiple sources like network ports, file sources, live camera sources
and the data needs to be muxed/decoded and sent to different sink elements. This would help in some parts of pipeline still working, 
even though one of the sources gets stalled which occurs more often in live sources. 


