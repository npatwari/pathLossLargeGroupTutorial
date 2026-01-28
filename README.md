# Path Loss Measurements using the Shout Measurement Framework
Neal Patwari and Kirk Webb, 28 Jan 2026

### Learning objectives

- Be able to use the Shout measurement framework to automate TX/RX functions across multiple POWDER nodes
- Work with complex-valued signals in the time domain and frequency domain
- Compute the received power from a received complex-valued signal
- Know how received power decreases as a function of path length

#### Group resource assignments table

Refer to this table when instructed to do so in the steps below.  Be sure to look at the correct entry for your group!

| Group # | Node Type | Nodes | Frequency Min | Frequency Max | Center Frequency | TX/RX Gains |
|---------|----|--------------|--------|-------|------|---------|
| 1,2,3 | Rooftop | `cbrssdr1-bes`, `cbrssdr1-browning`, `cbrssdr1-fm`, `cbrssdr1-honors`, `cbrssdr1-hospital` | 3391 | 3395 | 3393 | TX: 27, RX:30 |
| 4,5,6 |  Dense Deployment | `cnode-ebc`, `cnode-guesthouse`, `cnode-mario`, `cnode-moran`, `cnode-ustar` | 3396 | 3400 | 3398 | TX: 80, RX: 70 |

All should clone the [pathLossLargeGroupTutorial](https://github.com/npatwari/pathLossLargeGroupTutorial) git repo to their laptop.

## Instructor pre-class setup
**If you are using these instructions as part of an in-class tutorial, your instructor will have completed the steps in this section.** 

But if you are running these instructions on your own, you will need to do these steps yourself. 

### Instantiate a Shout experiment
Instantiate a `shout-long-measurement` profile experiment
* Navigate to: https://www.powderwireless.net/
* Go to Experiments: Start Experiment
* Click "Change Profile" and select "shout-long-measurement"
* Click "Next" to get to the "Parameterize" tab.
    - Use a compute node and orchestrator node type of d430
    - Use a Dataset to connect of "None"
    - Expand "CBAND X310 radios." and select the ones to be used. Expand the "Dense Site NUC+B210 radios to allocate." and select the nodes to be used. Of course, you are limited to [what is available](https://www.powderwireless.net/resinfo.php).
    - Expand "Frequency ranges for CBAND operation." You should look at [what is available](https://www.powderwireless.net/resinfo.php). Each group only needs a 2-3 MHz because the transmission is narrowband. 
    - Click "Next"
* On the "Finalize" step:
    - Select the "POWDER-Train-26" project, or whatever project that you and your students doing the tutorial are in.
* On the "Schedule" step, leave all fields at their defaults and click "Finish"
* Wait for the experiment to instantiate (turn green), and the startup scripts to finish

### Check each SDR's FPGA
Check each rooftop node `cbrssdr*` software-defined radio, which sometimes have an FPGA image different from the one we want for Shout, as follows.

Log on to each rooftop SDR's compute node. (I find it easiest to copy and paste the following commands and edit them in a separate editor, so I can copy and paste multiple ssh commands exactly as I want them.) For each compute node connected to a rooftop / `cbrssdr1` node, connect to it via ssh:
```
ssh -Y -p 22 -t username@pcXXX.emulab.net 'cd /local/repository/bin && tmux new-session -A -s shout &&  exec $SHELL'
```
where "username" is your username, and pcXXX is the compute **node** name.

When connected, run `uhd_usrp_probe`. 

If successful, it shows some information about the setup of the SDR.

If instead it shows:
> Error: RuntimeError: Expected FPGA compatibility number 38, but got 39:
> The FPGA image on your device is not compatible with this host code build. 

then do the following steps to flash the FPGA with the correct version of build:
1. Run: `/local/repository/bin/update_x310.sh`, which takes about 5 minutes.
2. When that is complete, Power Cycle the x310: Find the row in the List View of your POWDER experiment page with the name of your node "-x310". Using the Settings icon in that row (all the way to the right), select "Power Cycle" and click "Confirm".
3. After about a minute, run `uhd_find_devices` and `uhd_usrp_probe` to check that the radio is back on and now has the correct FPGA build version.

### Start the Shout Framework
Run this command into a new terminal window to connect via ssh to the orchestrator node.
```
ssh -Y -p 22 -t username@pcWWW.emulab.net 'cd /local/repository/bin && tmux new-session -A -s shout1 &&  exec $SHELL'
```
where `username` is your username, and replace pcWWW with the node name of the orchestrator. This command starts you in the `/local/repository/bin/` directory.

In the orchestrator window, run
```
./1.start_orch.sh
```
When it is done, on each compute node (attached to one of the SDRs), run
```
./2.start_client.sh
```
Now Shout is started and ready to "run" the measurement procedure.

## What Your Group Will Do
This section is completed by each group completing the tutorial. Split up the work as you wish among group members – suggestions for "(Team member X)" remind groups to split up the work.

### Instantiate a shout-iface-node Experiment
(Team member #1) 

Log in to Shout and start a [run-shout-measiface](https://www.powderwireless.net/instantiate.php?profile=1de3372e4a9f6c3ac872abc90d7f5c5636f76523) experiment. If the link doesn't work, got to Experiments: Start Experiment. Click the "Change Profile" button and search for and select "run-shout-measiface" from the "POWDER-Train-26" project. Click Next.

On the Parameterize screen, put into the "Node name of existing Shout orchestrator" field the node name of the orchestrator in your instructor's experiment. Ask if you're not sure. You don't need to change the Advanced options, but you can expand it to show that you are requesting a d430 compute node, no dataset, and to "Start X11 VNC" on this compute node.

Wait until the List View shows your node's status is "ready".

#### Edit the Experiment Parameters JSON File
(Team member #2) 

Either (a) open a X11 VNC window on your list view by using the pull down menu at the far right of the iface-node row; or (b) open a terminal window  ssh to your iface-node. 
```
ssh -Y -p 22 -t username@pcWWW.emulab.net 'cd /local/repository/bin && tmux new-session -A -s shout1 &&  exec $SHELL'
```
where `username` is your username, and replace pcWWW with the node name of the iface-node. This command starts me in the `/local/repository/bin/` directory.

You're going to need to edit a text file on your iface-node. People have their own preference for editor. I like `vi` but I know no one else who does. I suggest the `nano` editor. So run
```
nano /local/repository/etc/cmdfiles/save_iq_w_tx_cw.json
```
You're going to edit the entries of the JSON file parameter structure to have the following values:
| Field | Value it should be | Description |
|------|------|------|
| "cmd" | "save_iq_w_tx" | The name of command for the shout code to run. This saves the IQ data to file.|
| "sync" | true | True indicates Shout should synchronize transmission and reception | 
| "timeout" | 30 | How many ms before giving up at the TX |
| "nsamps" | 8192 | Number of samples to transmit |
| "wotxrepeat" | 0 | How many times the receiver should run when the TX is off (to capture noise samples)|
| "txrate" | 500e3 | Transmitter sampling rate|
|"txfreq"| (Use group setting)| Use the center frequency for your group. Because it is in MHz, write it with an `e6` at the end. For example, for 3387 MHz as a center frequency, write `3387e6`|
| "rxfreq"|  (Use group setting) | Use the exact same number as "txfreq" |
| "txgain"| `{"fixed": XX.0}` | Here `XX` is your group's gain for your SDR transmitter, either 27 or 80. This is due to the SDR in use: rooftop nodes use a NI X310, DD nodes use the NI B210. See the [hardware documentation](https://docs.powderwireless.net/hardware.html). |
| "txsamps"| `{"signal": "sine", "wfreq": 1e3, "wampl": 0.8}` | CW stands for continuous wave. This describes the sinusoidal wave in frequency (in Hz) and amplitude.|
| "txwait" | 3 | The transmitter is set to wait a certain duration (in ms, I believe) after the start of the second (when the PPS signal changes)|
| "rxrate" | 500e3 | Sampling rate at the receiver |
| "rxgain" | `{"fixed": YY.0}`| where `YY` is your gain for your SDR receiver, either 30 or 70, depending on your group (and thus what type of SDR you're using).|  
| "rxrepeat" | 2 | How many repetitions will be done of each link measurement |
| "rxwait" | `{"min": 50, "max": 2000, "res": "ms"}` | ?? |
| "txclients" | A list of the compute node IDs (in the ID column of List View) | The IDs are the names ending in "-comp" for Groups 1, 2 or 3, or ending in "-dd-b210" for Group 4,5 or 6. For example: `["cbrssdr1-honors-comp", "cbrssdr1-hospital-comp", "cbrssdr1-ustar-comp"]`|
| "rxclients"| The exact same list as for txclients| same as above

Save the file (`^O`), overwrite the same file by hitting Enter, and close the editor (`^X`).

Take a look at the file to double check. In particular check that 
* The rxfreq and txfreq are the same
* The txclients and rxclients are the same, and have strings that end in -comp or -dd-b210

```
cat /local/repository/etc/cmdfiles/save_iq_w_tx_cw.json
```

<!-- #### Change the shell script to call our JSON file
On the command line on your iface-node, change your directory using `cd /local/repository/bin/`.

Edit the `3.run_cmd.sh` script so that it calls on the save_iq_w_tx_cw.json file.
```
nano ./3.run_cmd.sh
```
Change the line that says `CMD="save_iq_w_tx_gold"` to instead say `CMD="save_iq_w_tx_cw"`.
Save the file (`^O`), overwrite the same file by hitting Enter, and close the editor (`^X`).
 -->
#### Execute the Shout measurement command
(Team member #3)

Two groups _can not_ run the Shout measurement command simultaneously.  [This spreadsheet](https://docs.google.com/spreadsheets/d/1fkBx37X2jsYdEbfvXn1atdOlv2okUsPXtIUUur662mg/edit?usp=sharing) will serve as a kind of multiple access control for us. Go to the [sheet](https://docs.google.com/spreadsheets/d/1fkBx37X2jsYdEbfvXn1atdOlv2okUsPXtIUUur662mg/edit?usp=sharing), and see if there is another group currently running a Shout measurement command on your node set (either "rooftop nodes" or "dense deployment nodes"). If so, wait. If not, write your group number in the appropriate box (top white box for groups 1,2, or 3; bottom white box for groups 4,5, or 6).

On your iface-node terminal window, run
```
./3.run_cmd.sh
```
This takes about 10 minutes. 

When your group is done, **please remember to delete your number** from the [spreadsheet](https://docs.google.com/spreadsheets/d/1fkBx37X2jsYdEbfvXn1atdOlv2okUsPXtIUUur662mg/edit?usp=sharing) so that someone else can use it.

### Copy your data back to your local laptop
(Team member #4) 

While still on the iface-node terminal window, do a `ls /local/data/` to see your data directory. Copy the directory name. If there is more than one, you probably ran the measurement multiple times, and you can pick whichever one you want.

Back _on a terminal window on your local laptop_, run
```
scp -r username@pcWWW.emulab.net:/local/data/<Data dir> <local directory to save it in>
```
where, again, change username, and change pcWWW to the iface-node name; and change `<Data dir>` to the directory you saw when running the `ls /local/data/` command, and `<local git directory for pathLossTutorial>` to the folder on your local laptop where you're running the pathLossTutorial.

Zip this local directory and share it with everyone on the team.

## Analyze the Data
(All team members individually)

In this step, you will load and run a Jupyter notebook. The file, `CheckShoutData.ipynb` is in this repo. It is also a [public Google Colab notebook](https://colab.research.google.com/drive/1g0DbZOUlDLly6y1m5UzvOpGiR9h4DKFC?usp=sharing) if you prefer to run the notebook on the cloud.

You'll edit and save your .ipynb file, and turn it in to the Canvas assignment.
