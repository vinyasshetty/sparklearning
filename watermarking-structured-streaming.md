Now talking about watermarking is little tricky.It provides two advantages and based on the window and ouput mode.

In terms of window: Now as we saw earlier\(Window Chapter\) we are grouping records based on the window in which the "event time" the given records fall in.But in the output mode "update" ,we will keep information about all the earlier records also,sometimes we may not need and we may want to retain information only upto certain time and this can be done with watermark.

Till now i have told you that "append" output mode cannot have groupBy on the dataframes, but if we use watermark first and then use a groupBy operations on the same watermark column, spark allows this.Because now spark has a timeframe  till where it needs to keep track of previous result.



