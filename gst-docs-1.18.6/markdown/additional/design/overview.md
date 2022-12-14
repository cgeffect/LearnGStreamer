# Overview

This part gives an overview of the design of GStreamer with references
to the more detailed explanations of the different topics.

This document is intented for people that want to have a global overview
of the inner workings of GStreamer.

## Introduction

GStreamer is a set of libraries and plugins that can be used to
implement various multimedia applications ranging from desktop players,
audio/video recorders, multimedia servers, transcoders, etc.

Applications are built by constructing a pipeline composed of elements.
An element is an object that performs some action on a multimedia stream
such as:

- read a file
- decode or encode between formats
- capture from a hardware device
- render to a hardware device
- mix or multiplex multiple streams

Elements have input and output pads called sink and source pads in
GStreamer. An application links elements together on pads to construct a
pipeline. Below is an example of an ogg/vorbis playback pipeline.

```
+-----------------------------------------------------------+
|    ----------> downstream ------------------->            |
|                                                           |
| pipeline                                                  |
| +---------+   +----------+   +-----------+   +----------+ |
| | filesrc |   | oggdemux |   | vorbisdec |   | alsasink | |
| |        src-sink       src-sink        src-sink        | |
| +---------+   +----------+   +-----------+   +----------+ |
|                                                           |
|    <---------< upstream <-------------------<             |
+-----------------------------------------------------------+
```

The filesrc element reads data from a file on disk. The oggdemux element
demultiplexes the data and sends a compressed audio stream to the vorbisdec
element. The vorbisdec element decodes the compressed data and sends it
to the alsasink element. The alsasink element sends the samples to the
audio card for playback.

Downstream and upstream are the terms used to describe the direction in
the Pipeline. From source to sink is called "downstream" and "upstream"
is from sink to source. Dataflow always happens downstream.

The task of the application is to construct a pipeline as above using
existing elements. This is further explained in the pipeline building
topic.

The application does not have to manage any of the complexities of the
actual dataflow/decoding/conversions/synchronisation etc. but only calls
high level functions on the pipeline object such as PLAY/PAUSE/STOP.

The application also receives messages and notifications from the
pipeline such as metadata, warning, error and EOS messages.

If the application needs more control over the graph it is possible to
directly access the elements and pads in the pipeline.

## Design overview

GStreamer design goals include:

- Process large amounts of data quickly
- Allow fully multithreaded processing
- Ability to deal with multiple formats
- Synchronize different dataflows
- Ability to deal with multiple devices

The capabilities presented to the application depends on the number of
elements installed on the system and their functionality.

The GStreamer core is designed to be media agnostic but provides many
features to elements to describe media formats.

## Elements

The smallest building blocks in a pipeline are elements. An element
provides a number of pads which can be source or sinkpads. Sourcepads
provide data and sinkpads consume data. Below is an example of an ogg
demuxer element that has one pad that takes (sinks) data and two source
pads that produce data.

```
 +-----------+
 | oggdemux  |
 |          src0
sink        src1
 +-----------+
```

An element can be in four different states: `NULL`, `READY`, `PAUSED`,
`PLAYING`. In the `NULL` and `READY` state, the element is not processing any
data. In the `PLAYING` state it is processing data. The intermediate
PAUSED state is used to preroll data in the pipeline. A state change can
be performed with `gst_element_set_state()`.

An element always goes through all the intermediate state changes. This
means that when en element is in the `READY` state and is put to `PLAYING`,
it will first go through the intermediate `PAUSED` state.

An element state change to `PAUSED` will activate the pads of the element.
First the source pads are activated, then the sinkpads. When the pads
are activated, the pad activate function is called. Some pads will start
a thread (`GstTask`) or some other mechanism to start producing or
consuming data.

The `PAUSED` state is special as it is used to preroll data in the
pipeline. The purpose is to fill all connected elements in the pipeline
with data so that the subsequent `PLAYING` state change happens very
quickly. Some elements will therefore not complete the state change to
`PAUSED` before they have received enough data. Sink elements are required
to only complete the state change to `PAUSED` after receiving the first
data.

Normally the state changes of elements are coordinated by the pipeline
as explained in [states](additional/design/states.md).

Different categories of elements exist:

- *source elements*: these are elements that do not consume data but
only provide data for the pipeline.

- *sink elements*: these are elements that do not produce data but
renders data to an output device.

- *transform elements*: these elements transform an input stream in a
certain format into a stream of another format.
Encoder/decoder/converters are examples.

- *demuxer elements*: these elements parse a stream and produce several
output streams.

- *mixer/muxer elements*: combine several input streams into one output
stream.

Other categories of elements can be constructed (see [klass](additional/design/draft-klass.md)).

## Bins

A bin is an element subclass and acts as a container for other elements
so that multiple elements can be combined into one element.

A bin coordinates its children???s state changes as explained later. It
also distributes events and various other functionality to elements.

A bin can have its own source and sinkpads by ghostpadding one or more
of its children???s pads to itself.

Below is a picture of a bin with two elements. The sinkpad of one
element is ghostpadded to the bin.

```
 +---------------------------+
 | bin                       |
 |    +--------+   +-------+ |
 |    |        |   |       | |
 |  /sink     src-sink     | |
sink  +--------+   +-------+ |
 +---------------------------+
```

## Pipeline

A pipeline is a special bin subclass that provides the following
features to its children:

- Select and manage a global clock for all its children.
- Manage `running_time` based on the selected clock. Running\_time is
the elapsed time the pipeline spent in the `PLAYING` state and is used
for synchronisation.
- Manage latency in the pipeline.
- Provide means for elements to comunicate with the application by the
`GstBus`.
- Manage the global state of the elements such as Errors and
end-of-stream.

Normally the application creates one pipeline that will manage all the
elements in the application.

## Dataflow and buffers

GStreamer supports two possible types of dataflow, the push and pull
model. In the push model, an upstream element sends data to a downstream
element by calling a method on a sinkpad. In the pull model, a
downstream element requests data from an upstream element by calling a
method on a source pad.

The most common dataflow is the push model. The pull model can be used
in specific circumstances by demuxer elements. The pull model can also
be used by low latency audio applications.

The data passed between pads is encapsulated in Buffers. The buffer
contains pointers to the actual memory and also metadata describing the
memory. This metadata includes:

- timestamp of the data, this is the time instance at which the data
was captured or the time at which the data should be played back.

- offset of the data: a media specific offset, this could be samples
for audio or frames for video.

- the duration of the data in time.

- additional flags describing special properties of the data such as
discontinuities or delta units.

- additional arbitrary metadata

When an element whishes to send a buffer to another element is does this
using one of the pads that is linked to a pad of the other element. In
the push model, a buffer is pushed to the peer pad with
`gst_pad_push()`. In the pull model, a buffer is pulled from the peer
with the `gst_pad_pull_range()` function.

Before an element pushes out a buffer, it should make sure that the peer
element can understand the buffer contents. It does this by querying the
peer element for the supported formats and by selecting a suitable
common format. The selected format is then first sent to the peer
element with a CAPS event before pushing the buffer (see
[negotiation](additional/design/negotiation.md)).

When an element pad receives a CAPS event, it has to check if it
understand the media type. The element must refuse following buffers if
the media type preceding it was not accepted.

Both `gst_pad_push()` and `gst_pad_pull_range()` have a return value
indicating whether the operation succeeded. An error code means that no
more data should be sent to that pad. A source element that initiates
the data flow in a thread typically pauses the producing thread when
this happens.

A buffer can be created with `gst_buffer_new()` or by requesting a
usable buffer from a buffer pool using
`gst_buffer_pool_acquire_buffer()`. Using the second method, it is
possible for the peer element to implement a custom buffer allocation
algorithm.

The process of selecting a media type is called caps negotiation.

## Caps

A media type (Caps) is described using a generic list of key/value
pairs. The key is a string and the value can be a single/list/range of
int/float/string.

Caps that have no ranges/list or other variable parts are said to be
fixed and can be used to put on a buffer.

Caps with variables in them are used to describe possible media types
that can be handled by a pad.

## Dataflow and events

Parallel to the dataflow is a flow of events. Unlike the buffers, events
can pass both upstream and downstream. Some events only travel upstream
others only downstream.

The events are used to denote special conditions in the dataflow such as
EOS or to inform plugins of special events such as flushing or seeking.

Some events must be serialized with the buffer flow, others don???t.
Serialized events are inserted between the buffers. Non serialized
events jump in front of any buffers current being processed.

An example of a serialized event is a TAG event that is inserted between
buffers to mark metadata for those buffers.

An example of a non serialized event is the FLUSH event.

## Pipeline construction

The application starts by creating a Pipeline element using
`gst_pipeline_new()`. Elements are added to and removed from the
pipeline with `gst_bin_add()` and `gst_bin_remove()`.

After adding the elements, the pads of an element can be retrieved with
`gst_element_get_pad()`. Pads can then be linked together with
`gst_pad_link()`.

Some elements create new pads when actual dataflow is happening in the
pipeline. With `g_signal_connect()` one can receive a notification when
an element has created a pad. These new pads can then be linked to other
unlinked pads.

Some elements cannot be linked together because they operate on
different incompatible data types. The possible datatypes a pad can
provide or consume can be retrieved with `gst_pad_get_caps()`.

Below is a simple mp3 playback pipeline that we constructed. We will use
this pipeline in further examples.

```
+-------------------------------------------+
| pipeline                                  |
| +---------+   +----------+   +----------+ |
| | filesrc |   | mp3dec   |   | alsasink | |
| |        src-sink       src-sink        | |
| +---------+   +----------+   +----------+ |
+-------------------------------------------+
```

## Pipeline clock

One of the important functions of the pipeline is to select a global
clock for all the elements in the pipeline.

The purpose of the clock is to provide a stricly increasing value at the
rate of one `GST_SECOND` per second. Clock values are expressed in
nanoseconds. Elements use the clock time to synchronize the playback of
data.

Before the pipeline is set to `PLAYING`, the pipeline asks each element if
they can provide a clock. The clock is selected in the following order:

- If the application selected a clock, use that one.

- If a source element provides a clock, use that clock.

- Select a clock from any other element that provides a clock, start
with the sinks.

- If no element provides a clock a default system clock is used for
the pipeline.

In a typical playback pipeline this algorithm will select the clock
provided by a sink element such as an audio sink.

In capture pipelines, this will typically select the clock of the data
producer, which in most cases can not control the rate at which it
produces data.

## Pipeline states

When all the pads are linked and signals have been connected, the
pipeline can be put in the `PAUSED` state to start dataflow.

When a bin (and hence a pipeline) performs a state change, it will
change the state of all its children. The pipeline will change the state
of its children from the sink elements to the source elements, this to
make sure that no upstream element produces data to an element that is
not yet ready to accept it.

In the mp3 playback pipeline, the state of the elements is changed in
the order alsasink, mp3dec, filesrc.

All intermediate states are traversed for each element resulting in the
following chain of state changes:

* alsasink to `READY`:  the audio device is probed

* mp3dec to `READY`:    nothing happens

* filesrc to `READY`:   the file is probed

* alsasink to `PAUSED`: the audio device is opened. alsasink is a sink and returns `ASYNC` because it did not receive data yet

* mp3dec to `PAUSED`:   the decoding library is initialized

* filesrc to `PAUSED`:  the file is opened and a thread is started to push data to mp3dec

At this point data flows from filesrc to mp3dec and alsasink. Since
mp3dec is `PAUSED`, it accepts the data from filesrc on the sinkpad and
starts decoding the compressed data to raw audio samples.

The mp3 decoder figures out the samplerate, the number of channels and
other audio properties of the raw audio samples and sends out a caps
event with the media type.

Alsasink then receives the caps event, inspects the caps and
reconfigures itself to process the media type.

mp3dec then puts the decoded samples into a Buffer and pushes this
buffer to the next element.

Alsasink receives the buffer with samples. Since it received the first
buffer of samples, it completes the state change to the PAUSED state. At
this point the pipeline is prerolled and all elements have samples.
Alsasink is now also capable of providing a clock to the pipeline.

Since alsasink is now in the `PAUSED` state it blocks while receiving the
first buffer. This effectively blocks both mp3dec and filesrc in their
`gst_pad_push()`.

Since all elements now return `SUCCESS` from the
`gst_element_get_state()` function, the pipeline can be put in the
`PLAYING` state.

Before going to `PLAYING`, the pipeline select a clock and samples the
current time of the clock. This is the `base_time`. It then distributes
this time to all elements. Elements can then synchronize against the
clock using the buffer `running_time`
`base_time` (See also [synchronisation](additional/design/synchronisation.md)).

The following chain of state changes then takes place:

* alsasink to `PLAYING`:  the samples are played to the audio device

* mp3dec to `PLAYING`:    nothing happens

* filesrc to `PLAYING`:   nothing happens

## Pipeline status

The pipeline informs the application of any special events that occur in
the pipeline with the bus. The bus is an object that the pipeline
provides and that can be retrieved with `gst_pipeline_get_bus()`.

The bus can be polled or added to the glib mainloop.

The bus is distributed to all elements added to the pipeline. The
elements use the bus to post messages on. Various message types exist
such as `ERRORS`, `WARNINGS`, `EOS`, `STATE_CHANGED`, etc..

The pipeline handles `EOS` messages received from elements in a special
way. It will only forward the message to the application when all sink
elements have posted an `EOS` message.

Other methods for obtaining the pipeline status include the Query
functionality that can be performed with `gst_element_query()` on the
pipeline. This type of query is useful for obtaining information about
the current position and total time of the pipeline. It can also be used
to query for the supported seeking formats and ranges.

## Pipeline EOS

When the source filter encounters the end of the stream, it sends an EOS
event to the peer element. This event will then travel downstream to all
of the connected elements to inform them of the EOS. The element is not
supposed to accept any more data after receiving an EOS event on a
sinkpad.

The element providing the streaming thread stops sending data after
sending the `EOS` event.

The EOS event will eventually arrive in the sink element. The sink will
then post an `EOS` message on the bus to inform the pipeline that a
particular stream has finished. When all sinks have reported `EOS`, the
pipeline forwards the EOS message to the application. The `EOS` message is
only forwarded to the application in the `PLAYING` state.

When in `EOS`, the pipeline remains in the `PLAYING` state, it is the
applications responsability to `PAUSE` or `READY` the pipeline. The
application can also issue a seek, for example.

## Pipeline READY

When a running pipeline is set from the `PLAYING` to `READY` state, the
following actions occur in the pipeline:

* alsasink to `PAUSED`:  alsasink blocks and completes the state change on the
next sample. If the element was `EOS`, it does not wait for a sample to complete
the state change.
* mp3dec to `PAUSED`:    nothing
* filesrc to `PAUSED`:   nothing

Going to the intermediate `PAUSED` state will block all elements in the
`_push()` functions. This happens because the sink element blocks on the
first buffer it receives.

Some elements might be performing blocking operations in the `PLAYING`
state that must be unblocked when they go into the PAUSED state. This
makes sure that the state change happens very fast.

In the next `PAUSED` to `READY` state change the pipeline has to shut down
and all streaming threads must stop sending data. This happens in the
following sequence:

* alsasink to `READY`:   alsasink unblocks from the `_chain()` function and returns
a `FLUSHING` return value to the peer element. The sinkpad is deactivated and
becomes unusable for sending more data.
* mp3dec to `READY`:     the pads are deactivated and the state change completes
when mp3dec leaves its `_chain()` function.
* filesrc to `READY`:    the pads are deactivated and the thread is paused.

The upstream elements finish their `_chain()` function because the
downstream element returned an error code (`FLUSHING`) from the `_push()`
functions. These error codes are eventually returned to the element that
started the streaming thread (filesrc), which pauses the thread and
completes the state change.

This sequence of events ensure that all elements are unblocked and all
streaming threads stopped.

## Pipeline seeking

Seeking in the pipeline requires a very specific order of operations to
make sure that the elements remain synchronized and that the seek is
performed with a minimal amount of latency.

An application issues a seek event on the pipeline using
`gst_element_send_event()` on the pipeline element. The event can be a
seek event in any of the formats supported by the elements.

The pipeline first pauses the pipeline to speed up the seek operations.

The pipeline then issues the seek event to all sink elements. The sink
then forwards the seek event upstream until some element can perform the
seek operation, which is typically the source or demuxer element. All
intermediate elements can transform the requested seek offset to another
format, this way a decoder element can transform a seek to a frame
number to a timestamp, for example.

When the seek event reaches an element that will perform the seek
operation, that element performs the following steps.

1) send a `FLUSH_START` event to all downstream and upstream peer elements.
2) make sure the streaming thread is not running. The streaming thread will
   always stop because of step 1).
3) perform the seek operation
4) send a `FLUSH` done event to all downstream and upstream peer elements.
5) send `SEGMENT` event to inform all elements of the new position and to complete
   the seek.

In step 1) all downstream elements have to return from any blocking
operations and have to refuse any further buffers or events different
from a `FLUSH` done.

The first step ensures that the streaming thread eventually unblocks and
that step 2) can be performed. At this point, dataflow is completely
stopped in the pipeline.

In step 3) the element performs the seek to the requested position.

In step 4) all peer elements are allowed to accept data again and
streaming can continue from the new position. A FLUSH done event is sent
to all the peer elements so that they accept new data again and restart
their streaming threads.

Step 5) informs all elements of the new position in the stream. After
that the event function returns back to the application. and the
streaming threads start to produce new data.

Since the pipeline is still `PAUSED`, this will preroll the next media
sample in the sinks. The application can wait for this preroll to
complete by performing a `_get_state()` on the pipeline.

The last step in the seek operation is then to adjust the stream
`running_time` of the pipeline to 0 and to set the pipeline back to
`PLAYING`.

The sequence of events in our mp3 playback example.

```
                                   | a) seek on pipeline
                                   | b) PAUSE pipeline
+----------------------------------V--------+
| pipeline                         | c) seek on sink
| +---------+   +----------+   +---V------+ |
| | filesrc |   | mp3dec   |   | alsasink | |
| |        src-sink       src-sink        | |
| +---------+   +----------+   +----|-----+ |
+-----------------------------------|-------+
           <------------------------+
                 d) seek travels upstream

    --------------------------> 1) FLUSH event
    | 2) stop streaming
    | 3) perform seek
    --------------------------> 4) FLUSH done event
    --------------------------> 5) SEGMENT event

    | e) update running_time to 0
    | f) PLAY pipeline
```
