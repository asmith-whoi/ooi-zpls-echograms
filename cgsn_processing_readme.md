## Remote processing of ZPLSC/G for CGSN instruments

Get the processing code at:
[https://github.com/oceanobservatories/ooi-zpls-echograms](https://github.com/oceanobservatories/ooi-zpls-echograms) 

CGSN server: ooi-cgdatadelivery1.whoi.net

---
### Server setup
The processing code directory is `~/ooi-zpls-echograms`. 

This is currently a branch/fork of the Ocean Observatories repo, because there
are several modifications required to use the code on the cgsn server. These
are noted below.

The fork also includes a modified shell script, `batch_zplsc_whoi.sh`, with
specific changes for use on the CGSN server.

If ever necessary, follow the instructions in the repo's main README to
install/update the code  and set up the _echograms_ python environment.

#### _Setup Notes:_

>The conda installation on the CGSN server behaves differently than on my
local machine (Mac), and presumably others. It was necessary to modify both
the batch shell script and the main python script for everything to
work correctly.
>
>In the main python script, disable the importlib import statement (line 18)
and remove the call to importlib.files() on line 742 so that the Image.open()
call references only the image name. Subsequently, it was essential that
the working directory be the script directory so that python can find the
image file.
>
>The batch script must be edited to call `source activate echogram` (instead
of `conda activate echogram`), and to comment out the line above that one,
which calls `.` on conda.sh.
>
>I also modify the batch script to point to the XML file location (which we
know thanks to a little preparatory housekeeping), and to remove the code
which processes it from the command line inputs. This makes calling the
script from the command line a little simpler and easier to troubleshoot.
Finally, the `cd` command which sets the working directory must also be
changed to reflect the location of the script on the server.

---
### Data setup

The location of the raw data on the CGSN server is:
```/mnt/cg-data/raw/<platform_designator>/R000XX/<path_to_instrument>/DATA```

Verify the appropriate XML file is at top level of DATA – if not, find and
make a copy of it there.

To identify this file, first determine the deployment start  date. Working
backwards from there, find the last XML file before the deployment begins.
This is most easily accomplished on the
[raw data repo](https://rawdata.oceanobservatories.org/files/) using your web
browser.

Create the directory for the processed files:

```commandline
mkdir -p /mnt/cg-data/proc/<platform_designator>/r000XX/<path_to_instrument>/processed
```

---
### Processing steps

Navigate into the script directory (necessary for the workaround detailed in 
_**Setup Notes**_ above):
```commandline
cd ~/ooi-zpls-echograms/ooi_zpls_echograms
```

Start a new screen session:
```commandline
screen -S <session_name>
```

_Processing can take a long time. Using screen will allow you to disconnect
from the server while the process  continues remotely, and will prevent
many potential interruptions._

Run the batch script:
```commandline
../utilities/batch_zplsc_whoi.sh <site> <raw_data_dir> <proc_dir> "<start_date>" "<end_date>"
```

_Use the start and end dates for the whole deployment. The script will do all
the associated date math to break up the data into the correct week-long chunks._

#### _Example:_
```commandline
./batch_zplsc_whoi.sh ga02hypm_upper /mnt/cg-data/raw/GA02HYPM/R00003/instruments/ZPLSG_sn55073/DATA /mnt/cg-data/proc/GA02HYPM/R00003/instruments/ZPLSG_sn55073/processed "2016-10-29" "2018-01-11"
```

Observe the processing and wait for at least one full week to complete, making
sure there are no errors. Be patient, there may be long periods with no
on-screen indication that anything is happening.

Once satisfied that everything is working, type `CTRL-A d` to detach from
screen. You may now disconnect from the server.

---
### Finishing up

Reconnect to the server and reattach to the screen session:
```commandline
screen -r [session name]
```

_The session name is optional if it is the only one running. If you have
forgotten or do not know the session name, use `screen -list`._

Confirm the processing is complete and there are no errors visible. Type
`CTRL-A \ ` to end the screen session.

It's a good idea to take a look inside the processed data directories and
make sure they exist for the full deployment and contain all the expected
files.

It is also a good idea to open and inspect the echogram images. From your
local machine, use rsync to transfer the echograms to you:
```commandline
rsync -azvP --exclude='*.nc' ooiuser@ooi-cgdatadelivery1.whoi.net:/mnt/cg-data/proc/<platform_designator>/r000XX/<path_to_instrument>/processed <local_destination_dir>
```

It is not necessary to open and inspect the netCDF files. They are large and
numerous. If the echograms look good, then it's pretty safe to assume the
netCDF files are good, too.

---
### Common questions and issues

#### Empty weekly processed directories and recursion limit errors

This is usually caused by corrupt raw data files. Corrupt files can result
if the instrument lost power or was not shut down properly, so they will
often occur at the end of a deployment, but can occur at other times as well.

The code on the server has been modified to avoid the errors, but if it is
ever updated or reinstalled from Ocean Observatories, then the problem may
return.

In the original code, the error occurs at line 520 in the function call to
_process_file(). This function is inside a list comprehension statement, which
does not directly support exception handling.

Restructure the statement as a for loop, and incorporate try/except/continue
statements to catch the errors.

#### Processing a single day, or other custom intervals

In this case, don't use the batch processing script. Call the python script
directly, as follows:
```commandline
zpls_echogram.py -s <site> -d <raw_data_dir> -o <proc_dir> -dr <start_date> <stop_date> -zm AZFP -xf <xml_file>
```

#### Processing multiple deployments at once

The batch script is a powerful time saver, but can only be used for a single
deployment at a time.

To process multiple deployments, use or write an additional shell script which
calls the batch script repeatedly, substituting the specific parameters for
each individual deployment.

#### Local processing

You can, of course, install and run the processing package on your local
computer. For processing single deployments, this will likely be much faster
than using the server.

Be aware that you will also need a local copy of the raw data, which can be
many gigabytes and take considerable time to acquire from the raw data server.
Processed files will also require many gigabytes of space on your machine,
which will then have to be transferred up to the raw data server, typically
via some other intermediary server.

Nevertheless, it may still be a good option for processing single deployment
data at the time of recovery. Usually the raw data will be placed onto a
portable external hard drive along with all other instrument data before
syncing en masse to the raw data server. If you have access to that hard
drive during the data recovery process, then you can access the raw data
there, and then add the processed files when finished.

Note that you may still need to copy the raw data temporarily to your
internal hard drive. Though it may seem convenient to point to directories
on the external drive when calling the processing script, this is known to
cause errors. Despite this extra step, though, it is still much easier and
faster than wget commands to pull the data from the internet.

#### Processing on Mac

The processing code has been tested and used with good results on Mac, with
one exception. The shell script for batch processing was written for Linux,
and depends on the date utility. Linux distros use GNU date. Mac is a flavor
of BSD Unix and BSD date differs significantly.

Batch processing on Mac requires using or writing an alternate shell script
to account for the different date implementation.