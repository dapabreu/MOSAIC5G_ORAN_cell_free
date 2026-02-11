# T-Tracer:

**T-Tracer** is an event monitoring and analysis tool used in the OAI ecossystem, with the main objective of inspecting the internal functioning of the stack in the 5G system. The main use of this tool is to collect data and generated events in real-time, during runtime, from the gNB as well as the UE side.

The T-tracer architecture is based of **traces**, registering these events. These are then exported into temporary files that can be read by the tracer tools, replayed and analyzed offline.

This documentation goes over the attempts at integrating this tool in a 5G stack, and in our several testbeds.

# Errors

```
ticked_ttilog:c: data comes without previous tick
iqlog.c:47:_event: bad buffer size
```

This means the following:

- `ticked_ttilog`: Frame data logs are being made without previous data being recorded. In other words, there is a jump being made that the system cannot track - it shows up in the events that are being referenced ahead.
- `iqlog`: incoming IQ Samples data has an invalid size - i.e. events are trying to be processed but arrive corrupted, empty, or not aligned with what the tracer is expecting - also appears in events referenced in front.
1. **We started testing with an old build:**

This was done with the purpose of understanding if the problem was that we were using different builds between the RAN nodes.

For this we use the same build for everyone, `w10`. After building, we tested again and verified that the problem was not here, and the t-tracer continued to generate the same error, both for the gNB and for the UE.

3. **The GUI doesn't work because we are in a different environment than fullstack?**

Technically, thinking this is wrong, because in the documentation the examples are all with UEs and gNBs only and the entire network is not used.

3. **Is it just the GUI that doesn't work?**

Yes, in this case we use the command:

```jsx
./textlog -d extracted_T_messages.txt -no-gui -ON -full
```

After recording a considerable trace (10M lines for the transfer of IQ Samples), we were able to activate the `textlog` and see the events without any problem. At the same time, when we connect the GUI with exactly the same trace, it crashes almost instantly.

4. **If the problem is related to the GUI let's try on a system with a virtual interface**

In a local VM, I tried to install Ubuntu 24.04 LTS and run exactly the same trace, without positive results, so it is not a matter of whether we have a graphical interface.

Also, since the OAI for T-tracer uses an X11 server we tested different technologies within `MobaXTerm` but without any results.

5. **Could it be from the build options?**

I considered this option since the build options documentation states that, by compiling the following scopes, files related to X are created.

- `enbscope, uescope, nrscope: libforms/X`

After compiling these scopes, we tried again to see the events in the GUI but without any success.

6. **We connected the GUI, but without subscribing to the events**

In other words, in this case we try to turn on the t-tracer GUI, but we turn off all events with the `-OFF` flag, which means that the GUI does not receive any events and therefore nothing is displayed.

Furthermore, we went to the file responsible for the error, in this case `openairinterface5g/common/utils/T/tracer/logger/iqlog.c` and commented out `abort` so that the program does not close when this condition is met:

The Framework expects the `buffer` to contain a certain size. When it arrives, **Tracer** checks that the size value is not exactly that and does the `abort`.

```
  if (bsize != N_RB_UL * symbols_per_tti * 12 * 4) {
    printf("%s:%d:%s: bad buffer size\n", __FILE__, __LINE__, __FUNCTION__);
    //abort();
  }
```

We were able to verify that the events that generate errors are:

```jsx
turning ON UE_PHY_MEAS		- does not work
turning ON UE_PHY_PDSCH_IQ	- does not work
```

Any groups containing these elements will not run. Apart from that we don't see anything on the graph, and this can be explained by the warning that is issued right at the beginning with:

```jsx
xy_plot.c:38:paint: WARNING: PAINT xy plot: width (0) or height (0) is wrong, not painting
event.c:152:repack_event: WARNING: widget id 55 not found
```

Furthermore, by exploring each event in the `T_messages.txt` file, which is the DB file that defines which parameters each event has, we can see the following regarding the events that crash the system:

```
ID = UE_PHY_MEAS
    DESC = UE PHY measurements
    GROUP = ALL:PHY:GRAPHIC:HEAVY:UE
    FORMAT = int,eNB_ID : int,frame : int,subframe : int,rsrp : int,rssi : int,snr: int,rx_power: int,noise_power: int,w_cqi: int,freq_offset

ID = UE_PHY_PDSCH_IQ
    DESC = UE PDSCH received IQ data
    GROUP = ALL:PHY:GRAPHIC:HEAVY:UE
    FORMAT = int,eNB_ID : int,frame : int,subframe : int,nb_rb : int,N_RB_UL : int,symbols_per_tti : buffer,pusch_comp
```

These events need to be referenced directly in the `nr-uesoftmodem` code so we can investigate where they are being invoked through the commands:

```
sudo grep -r "T_UE_PHY_MEAS" ~/ue-standard-build/openairinterface5g/
grep -r "T_UE_PHY_PDSCH_IQ" ~/ue-standard-build/openairinterface5g/
```

And this results in:

`T_UE_PHY_MEAS`

```
/home/admin/ue-standard-build/openairinterface5g/openair1/SCHED_NR_UE/phy_procedures_nr_ue.c:      T(T_UE_PHY_MEAS,
/home/admin/ue-standard-build/openairinterface5g/openair1/SCHED_UE/phy_procedures_lte_ue.c:      T(T_UE_PHY_MEAS, T_INT(eNB_id), T_INT(proc->frame_rx%1024), T_INT(proc->subframe_rx),
/home/admin/ue-standard-build/openairinterface5g/cmake_targets/ran_build/build/common/utils/T/T_IDs.h:#define T_UE_PHY_MEAS T_ID(320)
```

`T_UE_PHY_PDSCH_IQ`

```
/home/admin/ue-standard-build/openairinterface5g/openair1/PHY/NR_UE_TRANSPORT/nr_dlsch_demodulation.c:  T(T_UE_PHY_PDSCH_IQ,
/home/admin/ue-standard-build/openairinterface5g/openair1/PHY/LTE_UE_TRANSPORT/dlsch_demodulation.c:  T(T_UE_PHY_PDSCH_IQ, T_INT(eNB_id), T_INT(frame%1024),
/home/admin/ue-standard-build/openairinterface5g/cmake_targets/ran_build/build/common/utils/T/T_IDs.h:#define T_UE_PHY_PDSCH_IQ T_ID(316)
```

If we follow the corresponding paths we can find the events:

`T_UE_PHY_MEAS`

```
#if T_TRACER
    if(nr_slot_rx == 0)
      T(T_UE_PHY_MEAS,
        T_INT(gNB_id),
        T_INT(ue->Mod_id),
        T_INT(proc->frame_rx % 1024),
        T_INT(nr_slot_rx),
        T_INT((int)(10 * log10(ue->measurements.rsrp[0]) - ue->rx_total_gain_dB)),
        T_INT((int)ue->measurements.rx_rssi_dBm[0]),
        T_INT((int)(ue->measurements.rx_power_avg_dB[0] - ue->measurements.n0_power_avg_dB)),
        T_INT((int)ue->measurements.rx_power_avg_dB[0]),
        T_INT((int)ue->measurements.n0_power_avg_dB),
        T_INT((int)ue->measurements.wideband_cqi_avg[0]));
#endif
  }
```

`T_UE_PHY_PDSCH_IQ`

```
#if T_TRACER
  T(T_UE_PHY_PDSCH_IQ,
    T_INT(gNB_id),
    T_INT(ue->Mod_id),
    T_INT(frame % 1024),
    T_INT(nr_slot_rx),
    T_INT(nb_rb_pdsch),
    T_INT(frame_parms->N_RB_UL),
    T_INT(frame_parms->symbols_per_slot),
    T_BUFFER(&rxdataF_comp[gNB_id][0], 2 * /* ulsch[UE_id]->harq_processes[harq_pid]->nb_rb */ frame_parms->N_RB_UL * 12 * 2));
#endif
```

Comparing with the event definition in `T_messages`:

```
ID = UE_PHY_MEAS
    DESC = UE PHY measurements
    GROUP = ALL:PHY:GRAPHIC:HEAVY:UE
    FORMAT = int,eNB_ID : int,frame : int,subframe : int,rsrp : int,rssi : int,snr: int,rx_power: int,noise_power: int,w_cqi: int,freq_offset

ID = UE_PHY_PDSCH_IQ
    DESC = UE PDSCH received IQ data
    GROUP = ALL:PHY:GRAPHIC:HEAVY:UE
    FORMAT = int,eNB_ID : int,frame : int,subframe : int,nb_rb : int,N_RB_UL : int,symbols_per_tti : buffer,pusch_comp
```

We were able to see that in both cases most of the variables do not have a direct correspondence or even make sense.

Another important issue is events with the suffix `TICK`. These events supposedly serve to mark the time and sequence of internal cycles of each component during the execution of the modem. They (supposedly) should be responsible for visually synchronizing the graphics:

```
grep -r "TICK" ~/ue-standard-build/openairinterface5g/
```

When running this command we see that none of the TICK events are being executed on files in the 5G build stack. Are TICK events even being sent at least? This may explain why, in addition to the errors, we have no graphs to draw.

Finally, when we redirect the trace we have, passing it from `raw` format to `txt` format and consulting the data we have in it, we can verify that there are no `TICK` events to be emitted. Additionally, events that generate errors appear with this data:

`T_UE_PHY_MEAS`

```
UE_PHY_MEAS eNB_ID 0 frame 0 
subframe 549 rsrp 0 rssi -2147483648 
snr -93 rx_power 0 noise_power 0 w_cqi 0 freq_offset -3
```

There are entries that do not make sense, like the `rssi`

`T_UE_PHY_PDSCH_IQ`

```
UE_PHY_PDSCH_IQ eNB_ID 0 frame 0 
subframe 377 nb_rb 0 N_RB_UL 5 
symbols_per_tti 106 pusch_comp {buffer size:14 [e0 13 00 00 ... 00 00 00 00 ]}
```

The buffer is completely invalid.

7. **Try Tracer, but with a 4G LTE build**

Build the softmodems with `./build_oai -UE -eNB` and run in separate terminals:

```
# For the eNB:
sudo ./lte-softmodem -O 
../../../ci-scripts/conf_files/enb.band7.25prb.rfsim.nos1.conf 
--rfsim

# For the UE:
sudo ./lte-uesoftmodem -O 
../../../ci-scripts/conf_files/lteue.usim-ci.conf 
--rfsim -C 2680000000 -r 25 --ue-rxgain 140 --ue-txgain 120
 --T_stdout 2
 
 # Third terminal: 
 ./ue -d ../T_messages.txt
```

The results are positive right *outside the box* with minimal configuration, with furthers contributes to this hypothesis.

# Create the new T_messages.txt

`T_messages.txt` is the DB file responsible for defining the structure of each event. These events are then referenced by conditional blocks (`ifdef`) directly within the build code, where data and parameters are sent to the T-tracer.

Whenever we change `T_messages` or do - https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/common/utils/T/DOC/T/howto_new_trace.md or else we have to run the build again, do `sudo make` in both T-tracer folders, and run again.

<aside>
💡Important: new traces cannot run with old `T_messages`, as each trace is associated with the `T_messages` used to run it.

</aside>

# Insert T-Tracer in the network

Using a container, inserting T-tracer into the network is more complicated, having to copy all the build files and insert them into a dockerfile. Since this tool can be run locally (regarding even UE or gNB) then what we can do is simply run the necessary files inside the `rfsim5g-oai-nr-ue` container or have a shared volume and run it there. When we are done, we can send the trace data back to the host.

- **Example:**

```
# Copy inside the container
docker cp T_messages.txt <nome-do-container-do-UE>:/root/

# Run the reverse command to then copy the trace out
docker cp <nome-do-container-do-UE>:/root/FICHEIRO .
```

It is also possible by just opening the correct port in the docker-compose and running the Tracer from the Command Line options.