# osmocom-compose
Docker compose for Osmocom for testing CSFB, NITB style with osmo-bsc, osmo-msc, osmo-mgw, osmo-hlr, osmo-stp and configuration walk through.

Additional docs can be found at: http://ftp.osmocom.org/docs/latest/

## Build and run

After cloning, pull osmocom submodule:
```
git submodule init
git submodule update
```

To build and run:
```
docker-compose build --build-arg USER=$USER
docker-compose up -d
```

These are the services that should be running:

```
$ docker-compose ps
           Name                         Command               State    Ports
----------------------------------------------------------------------------
osmocom-compose_build_1      bash                             Exit 0
osmocom-compose_osmo-bsc_1   /usr/bin/osmo-bsc -c /data ...   Up
osmocom-compose_osmo-hlr_1   /usr/bin/osmo-hlr                Up
osmocom-compose_osmo-mgw_1   /usr/bin/osmo-mgw                Up
osmocom-compose_osmo-msc_1   /usr/local/bin/osmo-msc          Up
osmocom-compose_osmo-stp_1   /usr/bin/osmo-stp -c /data ...   Up
```

## Logging
You can view or tail the logs of individual services using the built in docker logging subsystem where `SERVICE_NAME` is one of `osmo-bsc, osmo-msc, osmo-mgw, osmo-hlr, osmo-stp`

```
docker-compose logs -f $SERVICE_NAME
```

Any error in attach should appear in the HLR + MSC logs

```
docker-compose logs -f osmo-hlr osmo-msc
```

Any error in call setup should appear in MSC + BSC logs:

```
docker-compose logs -f osmo-msc osmo-bsc
```

## Configure IP network:

**config/osmo-mgw.cfg**
Set IP of Media GW advertised to BTS for call audio. This address must be routable from BTS network, or you will have calls with no audio.
```
mgcp
  rtp bind-ip 10.22.10.7
```


## Configure RAN

Configure the PLMN in the BSC config
**config/osmo-bsc.cfg**
```
  network country code 001
  mobile network code 01
```
Configure the PLMN in the MSC config:
**config/osmo-msc.cfg**
```
 network country code 001
 mobile network code 01
```

Setup BTS unit ID to match the one advertised by the BTS in Abis/OML bring up, and the desired Band + ARFCN, LAC, BSIC, neighbors:
**config/osmo-bts.cfg**
```
 bts 0
  ip.access unit_id 1500 0
  band GSM900  
  location_area_code 11
  base_station_id_code 63  
  si2quater neighbor-list add earfcn 1400 thresh-hi 20 thresh-lo 10 prio 5 qrxlv
  trx 0
   arfcn 122 
```

If all is well, you can connect to the OsmoBSC VTY to confirm the BTS is up and configured:

`$ telnet localhost 4242`
```
OsmoBSC> show network
BSC is on MCC-MNC 001-01 and has 1 BTS

  Encryption: A5/0
  NECI (TCH/H): 1
  Use TCH for Paging any: 0
  Handover: Off
  Current Channel Load:
             CCCH+SDCCH4:   0% (0/4)
                   TCH/F:   0% (0/5)
  Last RF Command:
  Last RF Lock Command:
```

```
OsmoBSC> show bts 0  
BTS 0 is of sysmobts type in band GSM900, has CI 0 LAC 11, BSIC 63 (NCC=7, BCC=7) and 1 TRX
  Description: (null)
  ARFCNs: 122
  System Information present: 0x000c087e, static: 0x00000000
  Early Classmark Sending: 2G forbidden, 3G allowed (forbidden by 2G bit)
  Unit ID: 1500/0/0, OML Stream ID 0xff
  NM State: Oper 'Enabled', Admin 'Unlocked', Avail 'OK'
  Paging: 0 pending requests, 50 free slots
  OML Link: (r=10.22.10.8:45108<->l=10.22.10.7:3002)
  OML Link state: connected 21 days 23 hours 10 min. 39 sec.
  Neighbor Cells: Manual/separate SI5, ARFCNs: 100 200 (2) SI5: 10 20 (2)
  Current Channel Load:
             CCCH+SDCCH4:   0% (0/4)
                   TCH/F:   0% (0/5)
  Channel Requests        : 10 total, 0 no channel
  Channel Failures        : 0 rf_failures, 0 rll failures
  BTS failures            : 0 OML, 0 RSL
```

## Configure SGs
Configure the MSC/VLR endpoint that the MME will connect to:

**config/osmo-msc.cfg**
```
sgs
 local-port 29118
 local-ip 0.0.0.0
 vlr-name osmcocom.msc.mnc001.mcc001.3gpp.net
```

You can connect to the OsmoMSC VTY to confirm the SGs association:

`$ telent localhost 4254`

```
OsmoMSC> show sgs-connections        
 r=192.168.32.119:29118<->l=10.22.10.73:29118
```

Setup BTS neighbor list to include the EARFCN for the 4G cell in this case 1400:

**config/osmo-bsc.cfg**
```
 bts 0
  si2quater neighbor-list add earfcn 1400 thresh-hi 20 thresh-lo 10 prio 5 qrxlv
```

## Configure Subscribers

In HLR add the subscribers and assign an MSISDN:

`$ telnet localhost 4258`

```
OsmoHLR> enable
OsmoHLR# subscriber imsi 001010000000070 create
% Created subscriber 001010000000070
    ID: 3
    IMSI: 001010000000070
    MSISDN: none
OsmoHLR# subscriber imsi 001010000000070 update msisdn 5101000070
% Updated subscriber IMSI='001010000000070' to MSISDN='5101000070'
```

You should now be able to attach the subscriber and confirm in the HLR:

```
OsmoHLR> show subscriber imsi 001010000000070
    ID: 3               
    IMSI: 001010000000070
    MSISDN: 5101000070  
    VLR number: MSC-00-00-00-00-00-00
    last LU seen: Wed Apr 17 21:24:24 2019 UTC
```

The subscriber should also appear in the MSC subscriber cache:
```
OsmoMSC> show subscriber cache
  Subscriber:
    Extension: 5101000070
    LAC: 17/0x11
    RAN: EUTRAN-SGs
    IMSI: 001010000000070
    TMSI: 7870D432
    Flags:
     IMSI detached:             false
     Conf. by radio contact:    true
     Subscr. data conf. by HLR: true
     Location conf. in HLR:     true
     Subscriber dormant:        false
     Received cancel locataion: false
     MS not reachable:          false
     LA allowed:                true
    Paging: not paging for 0 requests
    SGs-state: SGs-ASSOCIATED
    SGs-MME: mmec01.mmegi8001.mme.epc.mnc001.mcc001.3gppnetwork.org
    Use: 3 (3*SGs)
```

