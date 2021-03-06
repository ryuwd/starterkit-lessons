# HLT intro

{% objectives "Learning Objectives" %}

* Learn about the LHCb trigger.
* Learn how to run Moore from settings and from TCK.
* Getting started with writing your own trigger selection.

{% endobjectives %} 

The Run 2 LHCb trigger reduced the input rate event rate of approximately 30 MHz
to 12.5 kHz of events that are written to offline tape storage. The rate
reduction is achieved in three steps: L0, HLT1 and HLT2. L0 is implemented in
custom FPGAs, has a fixed latency of 4 μs and a maximum output rate of 1 MHz.
HLT stands for High Level Trigger.

HLT1 and HLT2 are implemented as software applications that run on the online
farm; HLT1 runs in real-time and writes events to the local harddrives of the
farm machines, while HLT2 uses the rest of the available CPU (100% when there is
no beam) to process the events written by HLT1. The evolution of the disk buffer is
shown in the figure below. Events accepted by HLT2 are sent to offline storage.

![Disk buffer in 2016](img/DiskBuffer2016.png)

In Run I, both the reconstructions and selections used by HLT1, HLT2 and offline
were very different. In Run 2 the reconstructions used in HLT2 and offline are
identical, while selections might still be different. To be fast, HLT1 runs a subset
of the HLT2 reconstruction, for example HLT1 provides only tracks with PT > 500 MeV
and only Muon particle identification. We will see the difference in speed in the following. 

A new feature in Run 2 is the so-called Turbo stream. Since the reconstruction
available in HLT2 is the same as the offline reconstruction, physics analyses
can be done with the candidates created in HLT2. If a line is configured to be a
Turbo line, all information on the candidates that it selects is stored in the
raw event. The rest of the raw event, like sub-detector raw banks, is discarded,
and cannot be recovered offline. The advantage of the Turbo stream is that
less data are written to tape and no CPU intensive offline processing is needed.
The output of the Turbo stream is almost identical to the output of a Stripping 
line which goes to mdst.

For Run 3 LHCb is removing the hardware trigger and the first stage of the software trigger will run at 30 MHz. The HLT is being developed in the [Real-Time-Analysis project](https://twiki.cern.ch/twiki/bin/viewauth/LHCb/RealTimeAnalysis).

### Add trigger information to your ntuple
In this section and the following, we will explain how to find out which trigger lines selected my signal.

Copy the following DaVinci script to get an ntuple.

```python
from Configurables import DaVinci, CondDB

stream = "Semileptonic"
stripping_line = "B2DMuNuX_D0"

from PhysConf.Selections import AutomaticData, SelectionSequence, TupleSelection
my_particles_KPi = AutomaticData( "/Event/{}/Phys/{}/Particles".format(stream, stripping_line) )

tupsel_KPi = TupleSelection (
    'D0_Kpi' , 
    [my_particles_KPi]   ,
    Decay = "([B- -> ^(D0 -> ^K- ^pi+) ^mu-]CC) || ([B+ -> ^(D0 -> ^K- ^pi+) ^mu+]CC)",
    Branches = {
        "B"  : "([B- -> (D0 -> K- pi+) mu-]CC) || ([B+ -> (D0 -> K- pi+) mu+]CC)",
        "D0"  : "([B- -> ^(D0 -> K- pi+) mu-]CC) || ([B+ -> ^(D0 -> K- pi+) mu+]CC)",
        "K"  : "([B- -> (D0 -> ^K- pi+) mu-]CC) || ([B+ -> (D0 -> ^K- pi+) mu+]CC)",
        "pi"  : "([B- -> (D0 -> K- ^pi+) mu-]CC) || ([B+ -> (D0 -> K- ^pi+) mu+]CC)",
        "mu"  : "([B- -> (D0 -> K- pi+) ^mu-]CC) || ([B+ -> (D0 -> K- pi+) ^mu+]CC)",
        },
    ToolList = [] ,
    )

# 1. To get information about the TCK
tupsel_KPi.addTupleTool("TupleToolEventInfo")
# 2. To get information about trigger decisions
ttt = tupsel_KPi.addTupleTool("TupleToolTrigger")
ttt.Verbose = True #Needed to get trigger decisions

DaVinci().UserAlgorithms = [SelectionSequence( 'SEQ_KPi' , tupsel_KPi ).sequence()]
DaVinci().DataType = "2018"
CondDB(LatestGlobalTagByDataType=DaVinci().DataType)
DaVinci().Simulation = False
DaVinci().InputType = 'DST'
DaVinci().Lumi = False

DaVinci().TupleFile = "ntuple.root"
DaVinci().EvtMax = 1000


from PhysConf.Filters import LoKi_Filters
DaVinci().EventPreFilters = [LoKi_Filters(
    STRIP_Code="HLT_PASS_RE('Stripping"+stripping_line+"Decision')"
).sequencer('PreFilter')]


# file from lb-run -c best LHCbDirac/prod dirac-dms-lfn-accessURL --Protocol=xroot /lhcb/LHCb/Collision18/SEMILEPTONIC.DST/00075559/0000/00075559_00002816_1.semileptonic.dst
from GaudiConf import IOHelper
IOHelper().inputFiles([
    'root://eoslhcb.cern.ch//eos/lhcb/grid/prod/lhcb/LHCb/Collision18/SEMILEPTONIC.DST/00075559/0000/00075559_00002816_1.semileptonic.dst'
], clear=True)
```
The important line are
```python
tupsel_KPi.addTupleTool("TupleToolEventInfo")
ttt = tupsel_KPi.addTupleTool("TupleToolTrigger")
ttt.Verbose = True #Needed to get trigger decisions
```
`TupleToolEventInfo` adds basic information about the event to your ntuple. Normally it is added by default. You will find two branches Hlt1TCK and Hlt2TCK. `TupleToolTrigger` adds the decisions of trigger lines. You will only find branches called L0Global, Hlt1Global and Hlt2Global. We first have to find out which trigger lines were available for these data.

{% callout "What is a TCK?" %}

The Trigger Configuration Key (TCK) stores the configuration of the HLT in a 
database.
All algorithms and their properties are defined in it.
The key is usually given as a hexadecimal number. The last 4 digits define the L0 TCK.
The first 4 digits define the HLT configuration. HLT1 TCKs start with 1, HLT2 TCKs start
with 2.

{% endcallout %}

{% challenge "Which TCKs were used to trigger these data?" %}

Find out what the hexadecimal presentation of the Hlt1 and Hlt2 TCK is.

{% endchallenge %}

### Exploring a TCK: List of trigger lines

To get a list of all available TCKs one can use TCKsh which is a python shell with predefined
functions to explore TCKs, do

```
# Use the latest available Run 2 Moore release
$ lb-run -c best Moore/v28r3p1 TCKsh
> listConfigurations()
```

The commands `listL0Channels(<TCK>)`, `listHlt1Lines(<TCK>)` or `listHlt2Lines(<TCK>)` show lists of the lines in L0, Hlt1 or Hlt2. If you replace list with get, a list with the lines is returned.

{% challenge "Compare HLT1 lines from Run1 and Run2" %}

Try to find out which HLT1 lines were available in Run 1 and which are now available in Run 2.

What are the names of the topological trigger lines in Run 1 and Run 2?

{% endchallenge %}

### Add trigger information to your ntuple, continued
Once you have a list of trigger lines, you can add this to `TupleToolTrigger`:
```python
triggerList = [
    #L0 TCKsh getL0Channels(0x11741801)
    "L0MuonDecision",
    "L0DiMuonDecision"
    "L0HadronDecision",
    "L0ElectronDecision",
    "L0PhotonDecision",
    #Hlt1 lines TCKsh getHlt1Lines(0x11741801) 
    "Hlt1TrackMVADecision",
    "Hlt1TwoTrackMVADecision",
    "Hlt1TrackMVATightDecision",
    "Hlt1TwoTrackMVATightDecision",
    "Hlt1TrackMuonDecision",
    "Hlt1TrackMuonMVADecision",
    #Hlt2 lines TCKsh getHlt2(0x21751801)
    "Hlt2Topo2BodyDecision",
    "Hlt2Topo3BodyDecision",
    "Hlt2Topo4BodyDecision",
    "Hlt2TopoMu2BodyDecision",
    "Hlt2TopoMu3BodyDecision",
    "Hlt2TopoMu4BodyDecision"
]
ttt.TriggerList = triggerList
```
Be aware you have to append `Decision` to the name of the trigger line.


### Add TISTOS information to your ntuple
For analysis purposes it is important to know if your signal candidate was part of the trigger decision or not as one has different efficiencies for both categories. The following categories exist [ [LHCb-PUB-2014-039](https://cds.cern.ch/record/1701134/files/LHCb-PUB-2014-039.pdf) ]:
1. Triggered On Signal (TOS): events for which the presence of the signal is sufficient to generate a positive trigger decision.
2. Triggered Independent of Signal (TIS): the “rest” of the event is sufficient to generate a positive trigger decision, where the rest of the event is defined through an operational procedure consisting in removing the signal and all detector hits belonging to it.
3. Triggered On Both (TOB): these are events that are neither TIS nor TOS; neither the presence of the signal alone nor the rest of the event alone are sufficient to generate a positive trigger decision, but rather both are necessary.

For a simple trigger candidate (e.g. Track), more than around 70% (depending on the subdetector) of the online reconstructed trigger candidate hits need to be contained within the set of all the hits from all offline reconstructed signal parts. For a composite candidate, the combination of all individual trigger candidates is compared to the set of offline candidates.

To add TISTOS information to your ntuple do 
```python
tttt = tupsel_KPi.addTupleTool("TupleToolTISTOS")
tttt.Verbose = True #To decisions from all trigger stages
tttt.TriggerList = triggerList
```

{% challenge "Understanding TISTOS" %}

Explain why a single particle cannot be TOS on the Hlt1TwoTrackMVA line.

{% endchallenge %}

### Exploring a TCK: Properties of trigger lines

If you want to get an overview of an Hlt line and its algorithms, you can do 

```
> dump(<TCK>, lines = <line_names_regex>, file="output.txt")
```

More advanced is to search for properties of a line. One example is to search for the prescale.
The prescale determines how often a line is executed, 1.0 means always, 0.0 never.
Type for example:

```
> listProperties(0x214a160f,".*Hlt2DiMuonJPsi.*","AcceptFraction")
...
> listProperties(0x214a160f,".*Hlt2DiMuonJPsiPreScaler.*","AcceptFraction")
```
The regex is needed to search through all algorithms connected to a line.


<!-- ## Old

### Run Moore from settings (Run1 + Run2)
The application of the software trigger is called Moore. Moore relies on the same
algorithms as are used in Brunel to run the reconstruction and in DaVinci to
select particle decays.


Let's start with a simple Moore script, we call it runMoore.py:

```python
from Configurables import Moore
# Define settings
Moore().ThresholdSettings = "Physics_pp_2018"
Moore().RemoveInputHltRawBanks = True
Moore().Split = ''
# A bit more output
from Gaudi.Configuration import INFO
Moore().EnableTimer = True
#Moore().OutputLevel = INFO
# Input data
from PRConfig import TestFileDB
# The following call configures input data, database tags and data type
TestFileDB.test_file_db["2017NB_L0Filt0x1707"].run(configurable=Moore())
Moore().DataType = "2018"
# Override the TCK in the ThresholdSettings to match the input data
from Configurables import HltConf
HltConf().setProp("L0TCK", '0x1707')
# Remove a line which accepts every event when run on this sample
HltConf().RemoveHlt1Lines = ["Hlt1MBNoBias"]

Moore().EvtMax = 1000

from Configurables import HltMonitoringConf
HltMonitoringConf().OutputFile = "histos.root"

print Moore()
```

Try to run it with
```
$ lb-run -c best Moore/v28r3p1 gaudirun.py runMoore.py | tee log.txt
```

The property split defines if HLT1, HLT2 or both are run. In the example above both are run.
Change `Moore().Split` to `'Hlt1'` and rerun.
To run HLT2 only, you have to change some settings and the input file has to be a file where
HLT1 has run on:

```python
...
Moore().RemoveInputHltRawBanks = False # Why?
Moore().Split = 'Hlt2'
...
TestFileDB.test_file_db["2017_Hlt1_0x11611709"].run(configurable=Moore())
HltConf().setProp("L0TCK", '0x1709')
...
```

Note: HLT2 needs to know about the decisions of trigger lines used in HLT1.
The decisions are decoded from the definitions in the HLT1 TCK. Therefore, HLT2 can only
read data which have been created when running Moore from TCK and not from settings.

{% challenge "Compare Hlt1 and Hlt2" %}

What is reduction factor of Hlt1? (Search how many events are accepted by `Hlt1Global`.)

What is reduction factor of Hlt2? (Search how many events are accepted by `Hlt2Global`.)

What is difference in run time of Hlt1 and Hlt2?

{% endchallenge %}

### Run Moore from TCK* (Run1 + Run2)

There are two ways to run Moore, from `ThresholdSettings` and from `TCK` (Trigger Configuration Key).
When you develop a trigger line, it is more convenient to run from ThresholdSettings. The TCK
is used when running the trigger on the online farm or in MC productions as it uniquely defines the settings.

Running from TCK has a few restrictions:
 1 The L0TCK defined in the TCK and the one in data have to match.
 2 The HltTCK might be incompatible with a Moore version if the properties of C++ algorithms changed.	

Here is an example script to run from an Hlt1 TCK.

```python
from Configurables import Moore
# Define settings
Moore().UseTCK = True
# You can check in TCKsh which TCKs exist and for which Moore versions they can be used.
Moore().InitialTCK = "0x117318A1"
Moore().Split = 'Hlt1'
Moore().RemoveInputHltRawBanks = True
# In the online farm Moore checks if the TCK in data and the configuration are the same.
# Here we disable it as we run a different TCK.
Moore().CheckOdin = False
Moore().outputFile = "TestTCK1.mdf"
# A bit more output
from Gaudi.Configuration import INFO
Moore().EnableTimer = True
#Moore().OutputLevel = INFO
# Input data
from PRConfig import TestFileDB
TestFileDB.test_file_db["2017NB_L0Filt0x18A1"].run(configurable=Moore())
Moore().DataType = "2018"
Moore().EvtMax = 1000
print Moore()
```

Run with
```
$ lb-run -c best Moore/v28r3p1 gaudirun.py runMoore_hlt1_tck.py | tee log_hlt1_tck.txt
```

To run HLT2 on the output data of the first stage, use the following script:

```python
from Configurables import Moore
# Define settings
Moore().UseTCK = True
Moore().InitialTCK = "0x217318A1"
Moore().DataType = "2018"
Moore().Split = 'Hlt2'
Moore().RemoveInputHltRawBanks = False
Moore().CheckOdin = False
Moore().EnableOutputStreaming = True
Moore().outputFile = "TestTCK2.mdf"
# A bit more output
from Gaudi.Configuration import INFO
Moore().EnableTimer = True
#Moore().OutputLevel = INFO
# Input data
Moore().DDDBtag = 'dddb-20150724'
Moore().CondDBtag = 'cond-20170724'
Moore().inputFiles = ["TestTCK1.mdf"]
Moore().EvtMax = 100
```

### Adapt an existing trigger line (Run1 + Run2)

HLT2 lines are similar to stripping lines. They combine basic particles to composite objects
and you apply selections to get a clean sample. The framework in which you write a trigger
line looks different to a stripping line but the underlying algorithms are the same.
Documentation is found [here](https://twiki.cern.ch/twiki/bin/view/LHCb/LHCbTrigger#Developing_Hlt2_lines).
There you also find information how to measure the efficiency or the output rate of a trigger line.

HLT2 lines are found in the Hlt gitlab project in the package Hlt2Lines, see [here](https://gitlab.cern.ch/lhcb/Hlt/tree/2018-patches/Hlt/Hlt2Lines/python/Hlt2Lines).

Their settings, i.e. the cut definitions, have to be defined in HltSettings package as well,
see [here](https://gitlab.cern.ch/lhcb/Hlt/tree/2018-patches/Hlt/HltSettings/python/HltSettings).

As a hands on, we will change the prescale of a line with a high rate and then reduce its rate with extra cuts.
First setup a Moore lb-dev project from the nightlies.

```
$ lb-dev -c x86_64-centos7-gcc62-opt Moore/v28r3p1
$ cd MooreDev_v28r3p1
$ git lb-use Hlt
$ git lb-checkout Hlt/2018-patches Hlt/HltSettings
$ git lb-checkout Hlt/2018-patches Hlt/Hlt2Lines
$ make
```
Go to `Hlt/HltSettings/python/HltSettings/DiMuon/DiMuon_pp_2018.py`, search for prescale and change the prescale of `Hlt2DiMuonJPsi` to 1.0.
Run Moore again and see if the rate of this line has increased.

The line is defined in `Hlt/Hlt2Lines/python/Hlt2Lines/DiMuon/Lines.py`. The cut properties appear in the dictionary under `JPsi`.
Add an entry with the key `MinProbNN` and set some value. If you search for `JPsi` in the file, you will find that the lines
uses `JpsiFilter` from `Stages.py`. As it is used by another line as well, you have to add the entry to `JPsiHighPT` as well.
JpsiFilter simply uses muon pairs as input.  Go to Stages.py  and adapt the code of the `Hlt2ParticleFilter` to filter on the
pid of the muons, to do that add `(MINTREE('mu-' == ABSID, PROBNNmu) > %(MinProbNN)s )`
to the cut string. Run Moore again and see if the rate of this line has now decreased. If you are happy, you can do
```
$ git lb-push Hlt myNewLine
```
and create a merge request.

For more complicated developments which require changing many files or concurrent development of several people,
we encourage to use a full checkout of the `Moore` and `Hlt` projects and to use vanilla git commands.
A user friendly setup for this is being developed under the name [trigger-dev](https://gitlab.cern.ch/lhcb-HLT/trigger-dev).
We encourage people to check it out and give feedback on the [issues page](https://gitlab.cern.ch/lhcb-HLT/trigger-dev/issues).

{% challenge "Convert a stripping line to a Hlt2 line" %}

1. Pick a stripping line and convert it to a HLT2 line.
2. Make it a Turbo line (You have to set the property Turbo to true and the name of the line has to end with Turbo).
3. Run a rate test to determine the rate of the line, [instructions](https://twiki.cern.ch/twiki/bin/view/LHCb/MooreRateTestExamples) are found here.

{% endchallenge %} -->
