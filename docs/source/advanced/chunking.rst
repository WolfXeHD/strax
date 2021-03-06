Strax data model
=================

Data type and kind
-------------------

All data lives in *data types*, such as `raw_records` or `peak_basics`. Each of these has a fixed numpy datatype.

If a single row of two data types refers to the same physical / logical thing, such as an event or a peak, we say those data types have the same `data_kind`.


The Laws of Chunking
---------------------
You shall obey them.

1. Each data row corresponds to a time interval. Time and (endtime or (dt and length)) are mandatory fields in all datatypes.
2. Strax handles data in chunks. A chunk is also an interval (containing rows of data which are individually intervals).
3. Suppose you have a chunk of some datatype reaching from [t0, t1), then

   a. It contains all and only data that starts >= t0 or ends <= t1;
   b. All data outside the chunk ends <= t0, or starts >= t1. (Remember intervals are half-open; the boundary cases are not ambiguous.)
   c. In particular, every data row lies completely in one chunk. No data whatsoever lies partially in more than one chunk. This means chunks cannot be split at arbitrary times.

4. Zero-duration data rows are not allowed. Zero-duration chunks are allowed, but they cannot contain data.


Incoming data
-------------
From the perspective of a plugin, all incoming data is time-synchronized and merged by kind. Specifically:

* Data of the same kind is merged into a single array. If you depend on `events`, `peaks` and `peak_basics`, you will get two arrays: `events` and `peaks`. The first will be the merged array of `peaks` and `peak_basics`.
* Data of different kinds are synchronized by time. Strax will fetch a chunks of the first kind (`events`), then fetch as much as needed from the second kind (`peaks`) until you have all peaks that end before or at exactly the same time as the last event.

This example is a bit odd: when loading data of multiple kinds that are contained in each other, e.g. events and peaks, you very often want to use a `LoopPlugin` rather than a straight-up Plugin.

Outgoing data
-------------
Plugins can chunk their output as they wish, including withholding some data until the next chunk is sent out. Of course this requires keeping state, which means you cannot parallelize: see the chunk boundary handling section later in this documentation.

Savers, too, are free to chunk their data as they like; for example, to create files of convenient sizes. This affects the chunks you get when loading or reprocessing data. If you don't want this, e.g. if the next plugin in line assumes a particular kind of chunking you want to preserve, set the attribute `rechunk_on_save = False`.


Sorted output requirement
--------------------------
Strax requires that all output is sorted by time inside chunks.

Additionally, most or all plugins will assume that incoming data is time-ordered between chunks. That is, a subsequent chunk should not contain any data that starts before an item from a previous chunk ends. Strax data must be either consist of disjoint things, or if there are overlaps, chunk boundaries must fall in places where gaps exist.

It would have been much harder to code an algorithm if you do not know when you have seen all input before a certain time. Essentially you would have to wait until the end of the run before you can process any data, which goes against the idea of processing your data as a stream.

If your plugin removes or adds items from the original incoming array, it must output a different *data kind*. For example, during the initial data reduction steps, we remove items from 'raw_records' to make a new data kind 'records'. Here we change data kind, even though the fields in the output data type are identical to the fields in the input data type.