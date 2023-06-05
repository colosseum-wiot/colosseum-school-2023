# O-RAN Assignment

In this assignment, we will demonstrate how to use the SCOPE [[1]](#cite1) and ColO-RAN [[2]](#cite2) components of OpenRAN Gym [[3]](#cite3) to develop O-RAN-compliant Artificial Intelligence (AI)/Machine Learning (ML) solutions, deploy them as xApps on the near-RT RAN Intelligent Controller (RIC), and test them on a softwarized Radio Access Network (RAN) deployed on the Colosseum wireless network emulator and controlled by the RIC. In our setup, we are going to use 3 nodes, representing one Base Station (BS), one user (UE) and the near-RT RIC deployed on the edge of the network.

## Make a reservation on Colosseum

1. Connect to Colosseum VPN (instructions [here](https://colosseumneu.freshdesk.com/support/solutions/articles/61000285824-cisco-anyconnect-remote-vpn-access)) and login to [Colosseum website](https://experiments.colosseum.net).
2. Make a reservation with three SRNs (see instructions on [Making a Reservation](https://colosseumneu.freshdesk.com/en/support/solutions/articles/61000253463-making-a-reservation-interactive-and-batch-mode-)). The order of the SRNs shall be the following: for the first two SRNs choose the `scope-with-e2` image, while for the third choose the `coloran-near-rt-ric-prebuilt` image. It is noted that the RIC should be allocated to the last SRN in the Colosseum reservation.

    <p align="center">
      <img src="images/order-SRNs.png" width="350" />
    </p>

3. On the reservation page, you can find the assigned SRNs/nodes and their hostnames by hovering over the nodes.
4. At your scheduled reservation time, open the terminals and ssh as `root` user into the assigned Colosseum SRNs: `ssh root@<srn-hostname>`. The password for the `scope-with-e2` container is `scope`, while for the `coloran-near-rt-ric-prebuilt` image, the password is `ChangeMe`. We will need four terminals. Two for the SCOPE BS, one for the SCOPE UE, and one for the RIC. In one of the terminals, run the following command to start a Colosseum Radio-frequency (RF) scenario through the Colosseum CLI API (see instructions [here](https://colosseumneu.freshdesk.com/en/support/solutions/articles/61000253397-colosseum-cli)): `colosseumcli rf start 1009 -c`. More information about the scenario can be found in [Test Scenario All Paths 0 dB (1009)](https://colosseumneu.freshdesk.com/support/solutions/articles/61000277641-test-scenario-all-paths-0-db-1009). When the scenario starts, an output similar to the following is returned (time is in UTC):


```
Scenario Start Time is 22:30:45
```

## Start the SCOPE Base Station

As a first step, on one of the two BS terminals **we start the SCOPE base station on the first node of the reservation** and pass the configuration of the base station through the `radio_interactive.conf` file. Note that the base station application will only start if this is the first node in the reservation, otherwise a UE application will start. In the SCOPE node, type the following commands which will take care of starting the base station and the core network applications.

```
cd radio_api/
python3 scope_start.py --config-file radio_interactive.conf
tmux a
```

## Start the SCOPE UEs

After the BS starts, we can start the UE, using the same steps as before. Run the following commands on the **terminal of the second SRN of your reservation**, under the image `scope-with-e2`.

```
cd radio_api/
python3 scope_start.py --config-file radio_interactive.conf
tmux a
```

## Start the ColO-RAN Near-RT RIC

Now we start the ColO-RAN near-RT RIC. **On the SRN with the ColO-RAN image**, and its respective terminal, type the following commands. You need to specify the network interface used by the RIC to communicate with the RAN (e.g., the col0 interface in Colosseum). These commands will build and start the Docker images of the ColO-RAN near-RT RIC.

```
cd ~/radio_code/colosseum-near-rt-ric/setup-scripts
./setup-ric.sh col0
```

First, base Docker images, used to build the RIC images, are imported. Then, the four images composing the near-RT RIC are built, and their IP addresses and ports (defined in the `setup-lib.sh` script) are configured. These images include:

- **e2term**: This is the endpoint of the E2 messages on the RIC.
- **e2mgr**: It manages the messages to/from the E2 interface.
- **e2rtmansim**: It leverages the RIC Message Router (RMR) protocol to route the E2 messages inside the RIC.
- **db**: It maintains a database of the RAN nodes connected to the RIC.

After the building process completes, the Docker images are initialized as containers on a Colosseum Standard Radio Node (SRN) and listen for incoming connections from RAN nodes implementing an E2 termination endpoint. The container logs can be optionally read through the `docker logs` command, e.g., `docker logs e2term -f` shows the logs of the E2 termination (`e2term`) container.

After the previous step is done (i.e., setting up the RIC), type 

```
ifconfig col0
```
and save (i.e., in a Text Editor) the ip address of the `col0` interface since we will need it in the next step.

## Connect the SCOPE Base Station to ColO-RAN Near-RT RIC

After the ColO-RAN neat-RT RIC is started following the previous steps, the SCOPE base station can be connected to it. To achieve this, the SCOPE base station runs an instance of the O-RAN E2 termination, which we adapted from the O-RAN Software Community (OSC) Distributed Unit (DU) implementation. Once connected to the near-RT RIC, this component can exchange messages with the xApps running within.

Specifically, the E2 termination of the base station can:

- Receive RIC Subscription messages from the xApps.
- Send periodic RAN Key Performance Measurements (KPMs) to the xApps through RIC Indication messages.
- Receive RIC Control messages from the xApps.
- Interface with the SCOPE Application Programming Interfaces (APIs) to modify the scheduling and slicing configurations of the base station based on the received xApp control.

To include the IP address of the ColO-RAN node (e.g., the col0 interface of the ColO-RAN node in Colosseum), we modify the `build_odu.sh` script. On the second terminal of the SCOPE BS, which corresponds to the first node of our reservation, we do the following:

```
cd ~/radio_code/colosseum-scope-e2
vim build_odu.sh
```

In `<insert-ric-ip-address-here>` type the output of the `ifconfig col0` of the previous step. To edit a file with `vim` we type `a` and we continue with editing the file.

```
export RIC_HOST="<insert-ric-ip-address-here>"
export RIC_PORT=36422
export INTERFACE_TO_RIC="col0"
```
Save the changes (press `esc` and then `:wq`) in the file above and initialize the E2 termination on the SCOPE base station by typing the following commands on the SCOPE node. To revoke any unwanted changes in the `build_odu.sh` script type `esc` and then `:q!`.

```
./build_odu.sh clean
./run_odu.sh
```

The E2 termination is first built through the `build_odu.sh` script, which also sets the IP address and port of the near-RT RIC to connect to, as well as the network interface used for the connection to the RIC. Then, the E2 termination can be started through the `run_odu.sh` script, which initializes the E2 termination and connects it to the near-RT RIC. The successful connection of base station and near-RT RIC can be verified by reading the logs of the e2term container on the ColO-RAN node (e.g., through the command docker `logs e2term -f`). This log shows the association messages between the RIC and the base station, together with the ID of the connected base stations (e.g., `gnb:311-048-02010501`).

A successful setup of the BS's ODU will result in the following output.

<p align="center">
  <img src="images/success_odu.png" width="400" height="300"/>
</p>

## Initialize a Sample xApp

As a first step, on the SRN terminal with the ColO-RAN image, we type `docker logs e2term -f`, stop the process (`Ctrl + C`) and search (`Ctrl + F`) for the `gnb:` ID entry. An example is the following: `gnb:311-048-02010501`. We save the GNB ID (i.e., in a Text Editor). 

After starting the near-RT RIC, and connecting the SCOPE base station to it, the sample xApp provided as part of ColO-RAN can be initialized. This can be done through the `setup-sample-xapp.sh` script by typing the following commands **on the ColO-RAN node** and the respective terminal. This script takes as input the ID of the RAN node the xApp subscribes to (e.g., the base station), which can be read in the logs of the ColO-RAN e2term Docker container, as described in the previous sections. It then builds the Docker image of the sample xApp, and starts it as a Docker container on the near-RT RIC.

```
cd ~/radio_code/colosseum-near-rt-ric/setup-scripts
./setup-sample-xapp.sh <GNB ID>
```
For example: `./setup-sample-xapp.sh gnb:311-048-02010501`.

After the xApp container (named, for instance sample-xapp-24) has been started, the xApp process can be run with the following command. By running this command, the xApp subscribes to the RAN node specified in the xApp build script above (through a RIC Subscription message), and triggers periodic reports (sent through RIC Indication messages) of RAN KPMs from the node.

```
docker exec -it sample-xapp-24 /home/sample-xapp/run_xapp.sh
```
 
 After performing these steps, the ColO-RAN sample xApp logs on file the KPMs received from the RAN node. Users can add custom intelligence (e.g., through AI/ML agents) to the xApp by modifying the template scripts in the `setup/sample-xapp/` directory, and rebuilding the xApp Docker image through the steps described in this section.

## Change the ODU Report Timer
In order to change the ODU report Timer, we first stop (`Ctrl + C`) the processes on both the DU side of the SCOPE BS and the near-RT RIC. Then, we can type the following commands on the SRN terminal with the ColO-RAN image:

```
 docker exec -it sample-xapp-24 /bin/bash
 sed -i '442s/std::string event_def = "250"/std::string event_def = "1000"/' src/xapp.cc
```
With the above changes, we modify the timescale at which the BS's ODU reports the RAN's metrics back to the RIC. The initial granularity was set to 250 ms, and now we modify it to 1000 ms. Finally, we build the xApp's code, as following `./build_xapp.sh clean`.

We restart the ODU with `./run_odu.sh` (on the second terminal of the SCOPE BS) and we restart the RIC with:
```
 cd ../sample-xapp
 ./run_xapp.sh
```

## Clean up

This concludes the Colosseum O-RAN assignment. After you are done with your experiments, it is a good practice to stop the RF scenario by running the following command from within one of the SRN containers and to terminate your reservation from the Colosseum portal:
- `colosseumcli rf stop`
- In all terminals, close your ssh connections by typing: `exit`
- Access the Colosseum portal and delete your reservation.


# References

<a id="cite1">[1]</a> https://openrangym.com/ran-frameworks/scope

<a id="cite2">[2]</a> https://openrangym.com/o-ran-frameworks/coloran

<a id="cite3">[3]</a> https://openrangym.com/