# SModelS – a tool for interpreting simplified-model results from the LHC

 <img src=https://smodels.github.io/images/banner720.png />

Wolfgang Waltenberger (ÖAW and Uni Vienna), for the SModelS collaboration.

PyHEP 2020 virtual conference, July 2020.

## Preparatory step -- installation:


```python
# smodels is registered at the python packaging index
!pip install smodels
```

    Requirement already satisfied: smodels in /home/walten/.local/lib/python3.8/site-packages (1.2.3)
    Requirement already satisfied: pyslha>=3.1.0 in /home/walten/.local/lib/python3.8/site-packages (from smodels) (3.2.4)
    Requirement already satisfied: numpy>=1.13.0 in /home/walten/.local/lib/python3.8/site-packages (from smodels) (1.19.0)
    Requirement already satisfied: argparse in /home/walten/.local/lib/python3.8/site-packages (from smodels) (1.4.0)
    Requirement already satisfied: requests>=2.0.0 in /usr/lib/python3/dist-packages (from smodels) (2.22.0)
    Requirement already satisfied: unum>=4.0.0 in /home/walten/.local/lib/python3.8/site-packages (from smodels) (4.1.4)
    Requirement already satisfied: docutils>=0.3 in /usr/lib/python3/dist-packages (from smodels) (0.16)
    Requirement already satisfied: scipy>=1.0.0 in /home/walten/.local/lib/python3.8/site-packages (from smodels) (1.4.1)
    Requirement already satisfied: tex2pix>=0.1.5 in /home/walten/.local/lib/python3.8/site-packages (from pyslha>=3.1.0->smodels) (0.3.1)


## Part 1: Problem statement

Q: What problem does SModelS try to address?

A: SModelS is a tool for BSM (Beyond the Standard Model) physics. The problem with BSM physics is that **we are confronted with an entire landscape of promising models**:

<img width=400px src=./hitoshi.png />

The above picture shows Hitoshi Murayamas' impression of the theory landscape. 
Many theories are shown. Most of them come with many free parameters.

These theories want to be **confronted with a large number of experimental results**:

<img width=350px src="https://atlas.web.cern.ch/Atlas/GROUPS/PHYSICS/PUBNOTES/ATL-PHYS-PUB-2020-013/fig_23.png" />
<img width=350px src="https://twiki.cern.ch/twiki/pub/CMSPublic/SUSYSummary2017/Moriond2017_BarPlot.png" />

SModelS is a tool that aims at confronting BSM theories with the slew of searches for new physics. We do not address
measurements (which may also constrain BSM physics). How we do this? SModelS has two main components: **a database of experimental results, and a "theory decomposer"**. Let's start with the database:

## Part 2: Simplified Models (recap)

To recap: a simplified model is an "unphyiscal" theoretical model with only a minimum BSM particle content,
fixed production modes, fixed branching ratios, and with the signal strength being a free parameter of the model.

![sms1.png](attachment:sms1.png)

![sms2.png](attachment:sms2.png)

## Part 3: The Database


```python
from smodels.experiment.databaseObj import Database
```


```python
# instantiate a database. triggers a download from a CERN server.
# in this tutorial, you are free to use either our "unittest" database, 
# or the "official" v1.2.3 database. "official" is about 1 GB large,
# so if things are small, then use "unittest"
database = Database ( "unittest" ) ## "official"
print ( database )
```

    Database version: unittest123
    ------------------------------
    11 experimental results: 7 CMS, 4 ATLAS, 7 @ 8 TeV, 4 @ 13 TeV
    109 datasets, 536 txnames.
    



```python
## databases can of course also be constructed or augmented by the user. In its raw form, it consists of many 
## editable text files that get translated into a binary database by the framework. See e.g. here:
## https://smodels.readthedocs.io/en/stable/DatabaseDefinitions.html
## If needed, we can provide the tools that help in creating new entries.
```


```python
## lets find out how many results we find in the database. Some results are superseded (i.e. an update of the same
## analysis but with more data was published later), we filter them out.

len(database.getExpResults(useNonValidated=False, useSuperseded=False ))
```




    10




```python
## exp results can have multiple signal regions -- we call them datasets. Lets count the number of datasets in the
## database.
```


```python
sum([ len(x.datasets) for x in database.getExpResults( useSuperseded=False ) ])
```




    108




```python
## finally, per signal region we may have results for several simplified models, lets count the total
## number of efficiency maps
```


```python
sum([ sum([ len(y.txnameList) for y in x.datasets ]) for x in database.getExpResults( useSuperseded=False ) ])
```




    535




```python
## we have two types of results, upper-limit type (upper limits on production cross sections) and efficiency-map 
## type results. For upper-limit type results we sometimes have expected upper limits in addition to 
## observed upper limits. For such cases, as well as for effiency-map type results, we can contruct likelhoods.

## Let's count the number of "maps" for which we have likelihoods
```


```python
sum([ sum([ len([z for z in y.txnameList if z.hasLikelihood() ] ) for y in x.datasets ]) for x in database.getExpResults() ])
```




    515




```python
## thats difficult to read. let's do this slowly:

countLikelihoods = 0

for expRes in database.getExpResults():
    for ds in expRes.datasets:
        for txn in ds.txnameList:
            if txn.hasLikelihood():
                countLikelihoods += 1
                
print ( f"I am counting {countLikelihoods} likelihoods.")
```

    I am counting 515 likelihoods.


## Part 4: The Decomposer

The decomposer is tasked with decomposing a given theory into its simplified models spectrum, so the individual 
parts can be matched against the database:

<img src=https://smodels.readthedocs.io/en/stable/_images/decompScheme1b.png>


```python
# Import those parts of smodels that are needed for this exercise
# (We will assume the input is a SLHA file. For LHE files, use the lheDecomposer instead)
from smodels.theory import slhaDecomposer
from smodels.installation import installDirectory
from smodels.tools.physicsUnits import fb, GeV, TeV
```


```python
# Define the SLHA file name that contains the model we wish to decompose
filename="pyhep2020.slha"
```


```python
# Perform the decomposition. "elements" with predicted cross sections smaller than 0.5 fb
# are discarded. We perform "compression", a procedure that is described here:
# https://smodels.readthedocs.io/en/stable/Decomposition.html#compression-of-elements

listOfTopologies = slhaDecomposer.decompose(filename, sigcut = 0.5 * fb, 
                                            doCompress=True, doInvisible=True,minmassgap = 5* GeV)
```


```python
## mostly for reasons of efficiency, the simplified model "elements" are grouped into 
## simplified model "topologies". A topology just specifies the overall structure of a simplified model 
## (number of decays, number of particles emitted per decays), while an "element" specifies also the 
## final state Standard Model particle

# Print a summary of all the topologies generated, from simplest to most convoluted:
for top in listOfTopologies:
    print (top.describe())

```

    number of vertices: [0, 0], number of vertex particles: [[], []], number of elements: 1
    number of vertices: [0, 1], number of vertex particles: [[], [1]], number of elements: 1
    number of vertices: [0, 1], number of vertex particles: [[], [2]], number of elements: 2
    number of vertices: [1, 1], number of vertex particles: [[1], [1]], number of elements: 8
    number of vertices: [1, 1], number of vertex particles: [[1], [2]], number of elements: 10
    number of vertices: [1, 1], number of vertex particles: [[2], [2]], number of elements: 3
    number of vertices: [0, 2], number of vertex particles: [[], [2, 1]], number of elements: 9
    number of vertices: [1, 2], number of vertex particles: [[1], [1, 1]], number of elements: 28
    number of vertices: [1, 2], number of vertex particles: [[1], [1, 2]], number of elements: 4
    number of vertices: [1, 2], number of vertex particles: [[1], [2, 1]], number of elements: 50
    number of vertices: [1, 2], number of vertex particles: [[2], [1, 1]], number of elements: 14
    number of vertices: [1, 2], number of vertex particles: [[2], [1, 2]], number of elements: 4
    number of vertices: [1, 2], number of vertex particles: [[2], [2, 1]], number of elements: 20
    number of vertices: [2, 2], number of vertex particles: [[1, 1], [1, 1]], number of elements: 31
    number of vertices: [2, 2], number of vertex particles: [[1, 1], [1, 2]], number of elements: 8
    number of vertices: [2, 2], number of vertex particles: [[1, 1], [2, 1]], number of elements: 87
    number of vertices: [2, 2], number of vertex particles: [[1, 2], [2, 1]], number of elements: 42
    number of vertices: [2, 2], number of vertex particles: [[2, 1], [2, 1]], number of elements: 72
    number of vertices: [0, 3], number of vertex particles: [[], [1, 2, 1]], number of elements: 6
    number of vertices: [1, 3], number of vertex particles: [[1], [1, 2, 1]], number of elements: 37
    number of vertices: [1, 3], number of vertex particles: [[2], [1, 2, 1]], number of elements: 42
    number of vertices: [2, 3], number of vertex particles: [[1, 1], [1, 2, 1]], number of elements: 82
    number of vertices: [2, 3], number of vertex particles: [[2, 1], [1, 2, 1]], number of elements: 276



```python
## Let's have a look at the elements from the fourth topology, i.e. all "elements" with one decay on each of the
## two branches, one SM particle emerging from each decay:

# We can also print information for each element in the topology:
for element in listOfTopologies[3].elementList:
    print ('Element:',element.getParticles())
    print ('masses=',element.getMasses())
    print ('weight=',element.weight,'\n')
```

    Element: [[['W+']], [['W-']]]
    masses= [[2.69E+02 [GeV], 1.29E+02 [GeV]], [2.69E+02 [GeV], 1.29E+02 [GeV]]]
    weight= ['8.00E+00 [TeV]:5.96E-02 [pb]', '1.30E+01 [TeV]:1.45E-01 [pb]'] 
    
    Element: [[['W+']], [['Z']]]
    masses= [[2.69E+02 [GeV], 1.29E+02 [GeV]], [2.69E+02 [GeV], 1.29E+02 [GeV]]]
    weight= ['8.00E+00 [TeV]:1.32E-02 [pb]', '1.30E+01 [TeV]:2.62E-02 [pb]'] 
    
    Element: [[['W+']], [['higgs']]]
    masses= [[2.69E+02 [GeV], 1.29E+02 [GeV]], [2.69E+02 [GeV], 1.29E+02 [GeV]]]
    weight= ['8.00E+00 [TeV]:6.84E-02 [pb]', '1.30E+01 [TeV]:1.36E-01 [pb]'] 
    
    Element: [[['W-']], [['Z']]]
    masses= [[2.69E+02 [GeV], 1.29E+02 [GeV]], [2.69E+02 [GeV], 1.29E+02 [GeV]]]
    weight= ['8.00E+00 [TeV]:5.29E-03 [pb]', '1.30E+01 [TeV]:1.37E-02 [pb]'] 
    
    Element: [[['W-']], [['higgs']]]
    masses= [[2.69E+02 [GeV], 1.29E+02 [GeV]], [2.69E+02 [GeV], 1.29E+02 [GeV]]]
    weight= ['8.00E+00 [TeV]:2.74E-02 [pb]', '1.30E+01 [TeV]:7.11E-02 [pb]'] 
    
    Element: [[['higgs']], [['higgs']]]
    masses= [[2.69E+02 [GeV], 1.29E+02 [GeV]], [2.69E+02 [GeV], 1.29E+02 [GeV]]]
    weight= ['8.00E+00 [TeV]:5.63E-04 [pb]', '1.30E+01 [TeV]:1.85E-03 [pb]'] 
    
    Element: [[['q']], [['q']]]
    masses= [[9.91E+02 [GeV], 1.29E+02 [GeV]], [9.91E+02 [GeV], 1.29E+02 [GeV]]]
    weight= ['8.00E+00 [TeV]:1.36E-03 [pb]', '1.30E+01 [TeV]:7.31E-03 [pb]'] 
    
    Element: [[['q']], [['q']]]
    masses= [[9.91E+02 [GeV], 1.29E+02 [GeV]], [9.92E+02 [GeV], 1.29E+02 [GeV]]]
    weight= ['8.00E+00 [TeV]:4.15E-04 [pb]', '1.30E+01 [TeV]:2.27E-03 [pb]'] 
    


## Part 5: Putting things together

SModelS is a **python library**. However, SModelS also has **a few applications**, executables, 
that should cover many typical use cases:


```python
!runSModelS.py -h
```

    usage: runSModelS.py [-h] -f FILENAME [-p PARAMETERFILE] [-o OUTPUTDIR] [-d]
                         [-t] [-C] [-V] [-c] [-v VERBOSE] [-T TIMEOUT]
    
    Run SModelS over SLHA/LHE input files.
    
    optional arguments:
      -h, --help            show this help message and exit
      -f FILENAME, --filename FILENAME
                            name of SLHA or LHE input file or a directory path
                            (required argument). If a directory is given, loop
                            over all files in the directory
      -p PARAMETERFILE, --parameterFile PARAMETERFILE
                            name of parameter file, where most options are defined
                            (optional argument). If not set, use all parameters
                            from smodels/etc/parameters_default.ini
      -o OUTPUTDIR, --outputDir OUTPUTDIR
                            name of output directory (optional argument). The
                            default folder is: ./results/
      -d, --development     if set, SModelS will run in development mode and exit
                            if any errors are found.
      -t, --force_txt       force loading the text database
      -C, --colors          colored output
      -V, --version         show program's version number and exit
      -c, --run-crashreport
                            parse crash report file and use its contents for a
                            SModelS run. Supply the crash file simply via '--
                            filename myfile.crash'
      -v VERBOSE, --verbose VERBOSE
                            sets the verbosity level (debug, info, warning,
                            error). Default value is info.
      -T TIMEOUT, --timeout TIMEOUT
                            define a limit on the running time (in secs).If not
                            set, run without a time limit. If a directory is given
                            as input, the timeout will be applied for each
                            individual file.


However, since this is a python conference, we focus on SModelS as a library, a module.


```python
## Remember, we had a list of experimental results stored in "listOfResp", 
## and the decomposed elements of the model stored in "listOfTopologies". Now lets match them!

from smodels.theory.theoryPrediction import theoryPredictionsFor
```


```python
# Compute the theory predictions for each experimental result and print them:
print("\n Theory Predictions and Constraints:")
rmax = 0.
bestResult = None
## use only the first ten results, so we dont get lost
for expResult in database.getExpResults()[:10]:
    predictions = theoryPredictionsFor(expResult, listOfTopologies )
    if not predictions: continue # Skip if there are no constraints from this result
    print('\n %s (%i TeV)' %(expResult.globalInfo.id,expResult.globalInfo.sqrts.asNumber(TeV)))
    for theoryPrediction in predictions:
        dataset = theoryPrediction.dataset
        datasetID = dataset.dataInfo.dataId            
        mass = theoryPrediction.mass
        txnames = [str(txname) for txname in theoryPrediction.txnames]
        PIDs =  theoryPrediction.PIDs         
        print( "------------------------" )
        print( "TxNames = ",txnames )  
        print( "Theory Prediction = ",theoryPrediction.xsection.value )  #Signal cross section
        # Get the corresponding upper limit:
        print( "UL for theory prediction = ",theoryPrediction.upperLimit )
        # Compute the r-value
        r = theoryPrediction.getRValue()
        print( "r = ",r )
        #Compute likelihhod and chi^2 for EM-type results:
        if dataset.dataInfo.dataType == 'efficiencyMap':
            theoryPrediction.computeStatistics()
            print( 'Chi2, likelihood=', theoryPrediction.chi2, theoryPrediction.likelihood )
        if r > rmax:
            rmax = r
            bestResult = expResult.globalInfo.id

# Print the most constraining experimental result
print( "\nThe largest r-value (theory/upper limit ratio) is ",rmax )
if rmax > 1.:
    print( "(The input model is likely excluded by %s)" %bestResult )
else:
    print( "(The input model is not excluded by the simplified model results)" )
```

    
     Theory Predictions and Constraints:
    
     CMS-PAS-SUS-15-002 (13 TeV)
    ------------------------
    TxNames =  ['T1']
    Theory Prediction =  4.46E-03 [pb]
    UL for theory prediction =  6.81E+01 [fb]
    r =  0.06546813917867421
    
     ATLAS-SUSY-2013-02 (8 TeV)
    ------------------------
    TxNames =  ['T1']
    Theory Prediction =  3.92E-04 [pb]
    UL for theory prediction =  1.08E+01 [fb]
    r =  0.03620691467464338
    ------------------------
    TxNames =  ['T6WW']
    Theory Prediction =  6.57E-03 [pb]
    UL for theory prediction =  1.72E+01 [fb]
    r =  0.3823100896692222
    ------------------------
    TxNames =  ['T2']
    Theory Prediction =  1.77E-03 [pb]
    UL for theory prediction =  6.10E+00 [fb]
    r =  0.29082248817957596
    ------------------------
    TxNames =  ['T5WW']
    Theory Prediction =  9.07E-03 [pb]
    UL for theory prediction =  3.23E+01 [fb]
    r =  0.28048687857303356
    
     ATLAS-SUSY-2013-12 (8 TeV)
    ------------------------
    TxNames =  ['TChiWZ']
    Theory Prediction =  1.85E-02 [pb]
    UL for theory prediction =  4.84E+02 [fb]
    r =  0.03813203554779975
    
     CMS-SUS-13-012 (8 TeV)
    ------------------------
    TxNames =  ['T1', 'T2']
    Theory Prediction =  1.36E-04 [pb]
    UL for theory prediction =  1.21E+00 [fb]
    r =  0.11260635791889409
    Chi2, likelihood= 0.6453736685170952 0.0036842875361852794
    
    The largest r-value (theory/upper limit ratio) is  0.3823100896692222
    (The input model is not excluded by the simplified model results)


*Excluded it is, and by quite a margin (r=6.85 >> 1). Another theorist's hope murdered viciously by ruthless experimentalists.*

## Part 6: show "missing topologies"

Produce some diagnostics about topologies that remained "unconstrained", i.e. do results for this topology exist
in principle, and we just fall outside of the mass grids?


```python
## import the coverage module for this task
from smodels.tools import coverage
```


```python
# Create missing Topologies object
uncovered = coverage.Uncovered(listOfTopologies)
```


```python
# Print basic information about coverage:
print ("Total missing topology cross section (fb): %10.3E" %(uncovered.getMissingXsec()))
print ("Total cross section where we are outside the mass grid (fb): %10.3E" %(uncovered.getOutOfGridXsec()))
print ("Total cross section in long cascade decays (fb): %10.3E" %(uncovered.getLongCascadeXsec()))
print ("Total cross section in decays with asymmetric branches (fb): %10.3E" %(uncovered.getAsymmetricXsec())) 
```

    Total missing topology cross section (fb):  2.325E+03
    Total cross section where we are outside the mass grid (fb):  2.337E+02
    Total cross section in long cascade decays (fb):  7.354E+02
    Total cross section in decays with asymmetric branches (fb):  1.557E+03



```python
# Get list of topologies which are not tested by any result:
missingTopos = uncovered.missingTopos
# print information about the first few missing topologies and
# the elements contributing to the topology: 
for top in missingTopos.topos[:5]:
    print ('\nmissing topology:',top.topo)
    print ('Contributing elements:')
    for el in sorted(top.contributingElements, key=lambda el: el.weight):
        print (el,', xsection:',el.weight)
```

    
    missing topology: [[],[]](MET,MET)
    Contributing elements:
    [[],[]] , xsection: ['8.00E+00 [TeV]:4.81E-04 [pb]', '1.30E+01 [TeV]:1.58E-03 [pb]']
    
    missing topology: [[],[[jet]]](MET,MET)
    Contributing elements:
    [[],[[q]]] , xsection: ['8.00E+00 [TeV]:6.15E-04 [pb]', '1.30E+01 [TeV]:3.15E-03 [pb]']
    
    missing topology: [[],[[jet,jet]]](MET,MET)
    Contributing elements:
    [[],[[c,c]]] , xsection: ['8.00E+00 [TeV]:9.63E-05 [pb]', '1.30E+01 [TeV]:5.47E-04 [pb]']
    [[],[[q,q]]] , xsection: ['8.00E+00 [TeV]:9.63E-05 [pb]', '1.30E+01 [TeV]:5.47E-04 [pb]']
    
    missing topology: [[[W]],[[W]]](MET,MET)
    Contributing elements:
    [[[W+]],[[W-]]] , xsection: ['8.00E+00 [TeV]:5.96E-02 [pb]', '1.30E+01 [TeV]:1.45E-01 [pb]']
    
    missing topology: [[[higgs]],[[higgs]]](MET,MET)
    Contributing elements:
    [[[higgs]],[[higgs]]] , xsection: ['8.00E+00 [TeV]:5.63E-04 [pb]', '1.30E+01 [TeV]:1.85E-03 [pb]']



```python
# Get list of topologies which are outside the upper limit and efficiency map grids:
outsideGrid = uncovered.outsideGrid
# print information about the first few missing topologies and
# the elements contributing to the topology: 
for top in outsideGrid.topos[:5]:
    print ('\noutside the grid topology:',top.topo)
    print ('Contributing elements:')
    for el in top.contributingElements:
        print (el,'mass=',el.getMasses())
```

    
    outside the grid topology: [[[W]],[[higgs]]](MET,MET)
    Contributing elements:
    [[[W+]],[[higgs]]] mass= [[2.69E+02 [GeV], 1.29E+02 [GeV]], [2.69E+02 [GeV], 1.29E+02 [GeV]]]
    [[[W-]],[[higgs]]] mass= [[2.69E+02 [GeV], 1.29E+02 [GeV]], [2.69E+02 [GeV], 1.29E+02 [GeV]]]
    
    outside the grid topology: [[[jet]],[[b,b]]](MET,MET)
    Contributing elements:
    [[[q]],[[b,b]]] mass= [[9.91E+02 [GeV], 1.29E+02 [GeV]], [8.65E+02 [GeV], 1.29E+02 [GeV]]]
    
    outside the grid topology: [[[jet]],[[jet,jet]]](MET,MET)
    Contributing elements:
    [[[q]],[[c,c]]] mass= [[9.91E+02 [GeV], 1.29E+02 [GeV]], [8.65E+02 [GeV], 1.29E+02 [GeV]]]
    [[[q]],[[q,q]]] mass= [[9.91E+02 [GeV], 1.29E+02 [GeV]], [8.65E+02 [GeV], 1.29E+02 [GeV]]]
    [[[q]],[[c,c]]] mass= [[9.92E+02 [GeV], 1.29E+02 [GeV]], [8.65E+02 [GeV], 1.29E+02 [GeV]]]
    [[[q]],[[q,q]]] mass= [[9.92E+02 [GeV], 1.29E+02 [GeV]], [8.65E+02 [GeV], 1.29E+02 [GeV]]]
    
    outside the grid topology: [[[jet]],[[t,t]]](MET,MET)
    Contributing elements:
    [[[q]],[[t+,t-]]] mass= [[9.91E+02 [GeV], 1.29E+02 [GeV]], [8.65E+02 [GeV], 1.29E+02 [GeV]]]
    
    outside the grid topology: [[[b],[W]],[[b],[W]]](MET,MET)
    Contributing elements:
    [[[b],[W+]],[[b],[W-]]] mass= [[8.78E+02 [GeV], 2.69E+02 [GeV], 1.29E+02 [GeV]], [8.78E+02 [GeV], 2.69E+02 [GeV], 1.29E+02 [GeV]]]


## Part 7: Future developments

**Developments we foresee to enter the next release:**

 * Support for scenarios with **long-lived particles**: so we can describe results with long-lived particles, like searches for displaced objects, disappearing tracks, etc

![beyondmet.png](attachment:beyondmet.png)

* **support for pyhf**: we will soon be able to make use of the likelihoods provided by pyhf (see separate talk)


![pyhf_prel.png](attachment:pyhf_prel.png)

* **combinations of likelihoods**: we aim at publications that describe which 
  analyses in the SModelS database can be considered to be uncorrelated and which cannot. 
  Uncorrelated analyses can then be combined via their likelihoods.

![matrix_aggressive.png](attachment:matrix_aggressive.png)

## Part 8: Links and References

**Here is a collection of links around SModelS:**

SModelS proper: https://smodels.github.io

Our repository: https://github.com/SModelS

More python and jupyter examples: https://smodels.readthedocs.io/en/stable/Examples.html

Documentation: https://readthedocs.org/projects/smodels/


```python

```
