
# Reproduction of the group Assignment MAT/BIO 494 Data Analysis in Neuroscience

## Reproduction Steps:

Using a dedicated Docker container we used python3 to programmatically downloaded a file `_hybrid_connectivity_matrix_20171103_092033.xlsx`  inside a script `qi_ascoli.py` (the file was named after the two authors, whose publications most informed our work[1][2]).

The code snippet for downloading those files (a snippet from qi_ascoli.py) is pasted below:
```
# Get some hippocampus connectivity data, based on a conversation with
# academic researchers on GH:
# https://github.com/Hippocampome-Org/GraphTheory/issues?q=is%3Aissue+is%3Aclosed
# scrape hippocamome connectivity data, that I intend to use to program neuromorphic hardware.
# conditionally get files if they don't exist.
path_xl = '_hybrid_connectivity_matrix_20171103_092033.xlsx'
if not os.path.exists(path_xl):
    os.system('wget https://github.com/Hippocampome-Org/GraphTheory/files/1657258/_hybrid_connectivity_matrix_20171103_092033.xlsx')
```

We estimated the rheobase current injection for the excitatory and inhibitory classes of cells using code from the _neuronunit_ model testing library which contains convience methods, which encapsulate a complex implementation of the rheobase search algorithm. _Neuronunits's_ rheobase search accesses _NEURON_ solvers, in a parallel manner. Parallel NEURON simulations are used to extract rheobase current injection values implied by a set of Izhikevich equations, by iteratively running an appropriate set of differential equations, that exhaustively search an appropriate range of current injection values.

This code was subequently commented out, but it persists in that form at the top of `qi_ascoli.py`. A snippet is below:
```
from neuronunit.models.reduced import ReducedModel
from neuronunit.optimization import get_neab
model = ReducedModel(get_neab.LEMS_MODEL_PATH,name=str('vanilla'),backend=('NEURON'))
attrs = {'a':0.02, 'b':0.2, 'c':-65+15*0.5, 'd':8-0.5**2 }

from neuronunit.tests import fi
model.set_attrs(attrs)
from neuronunit.optimization import get_neab
rtp = get_neab.tests[0]
rheobase0 = rtp.generate_prediction(model)
model = None

attrs2 = {'a':0.02+0.08*0.5, 'b':0.2-0.05*0.5, 'c':-65, 'd':2 }

model.set_attrs(attrs2)
from neuronunit.optimization import get_neab
rtp = get_neab.tests[0]
rheobase1 = rtp.generate_prediction(model)
print(rheobase0['value'],rheobase1['value'])
assert rheobase0['value'] != rheobase1['value']
```

Once rheobase current injections for the excitatory and inhibitory populations of cells where found, we wired togethor the cell synapses. To do so we implemented wiring rules that handled four specific of sub-networks, between the two main classes of cells: Excitatory to Excitatory projections, Inhibitory to Excitatory, Excitatory to Inhibitory, and Inhibitory to Inhibitory, using the experimentally derived wiring rules implied by the spreadsheet: `_hybrid_connectivity_matrix_20171103_092033.xlsx`

PyNN code, for wiring the network, and recording the membrane potential was subsequently executed, inside the scope of a main method in qi_ascoli `sim_runner(wgf)`. Sim_runner takes a weight value as an argument, and it uniformly assigns this value to all synaptic weights in the simulated network model.

Another file, `forked.py` was used to invoke the python code in `qi_ascoli`, in an embarrassingly parallel manner, by using operating system level calls to the BASH command fork, making it possible to simulate very many networks of different weight values in a short amount of time.

Denise analysed the initial wiring map, in excel format (_hybrid_connectivity_matrix_20171103_092033.xlsx), and used it to extract indegree and outdegree distributions, for each cell in the network. Denise used excel to create vectors which describe indegree and outdegree of each neuron, as related to different anatomical regions in the hippocampus: 'DG', 'EC', 'CA1', 'CA2', and 'CA3'. She then used python to plot these region specific indegree and outdegree distributions per hippocampus sub-region. More information on Denises's workflow can be found in here jupyter notebook found at: https://github.com/russelljjarvis/DAnalysisCNeuro/blob/master/RichClub1-2.ipynb

Subsequently Daniel Petty's code for interactive network visualization using Shiny a module in R is launched. Daniel's R code acted on csv files defined in qi_ascoli. The initial wiring rules defined in `_hybrid_connectivity_matrix_20171103_092033.xlsx` used negative integers '-1', and '-2' to denote the presence of inhibitory synaptic connections, and '1' and '2' denoted excitatory connections.

We decomposed this connection matrix into fourt sub connection matrices that dealt with specific projections between the two populations, conventionally denoted by: 'EE', 'EI', 'II' and 'IE'. Such that '1' entry in the matrix denoted the presence of connection, and '0' denoted the absence of a connection. Values of '2', and '-2' in Ascoli's initial wiring map where used to represent putative connections, hypothesised connections, that have not been falsified yet. We upgraded putative connections to the status of confirmed connections, for the purposes of adding synaptic drive to neurons, that may otherwise suffer from sparse connectivity, making it easier to tip the neurons into a more realistic high conductance state.

The R package chorddiag was able to take those matrices and create chord diagrams. Getting the chord diagrams into a form we could repeatedly use independent of R necessitated the R package shiny.

The network visualization requires taking the excitatory to excitatory, excitatory to inhibitory, inhibitory to inhibitory, and inhibitory to excitatory connection matrices and turning them into graphs via the igraph package. The visNetwork package is interactive and much more versatile for visualization, but lacks the ability to directly translate the adjacency matrices. After converting the igraph graph into the visNetwork format, attributes are assigned to each node in the network: betweenness, group, location, and firing rate. Then the visNetwork plot is called in a shiny app, giving the visualization to be run independent of R.

We have tested the Dockerfile up to line 90, and we where able to confirm that this build is sufficient for launching both R, and python3 with PyNN, elephant and other dependencies, however we are unsure if running lines 91, and 92. Will flawlesly run the network visualization software. We are confident, that conceptually this approach to running all the software is correct.
https://github.com/russelljjarvis/DAnalysisCNeuro/blob/master/Dockerfile#L90-#L92
```
RUN R -e 'install.packages(c("rPython","shiny","igraph","visNetwork,"pracma,"stringr","chorddiag"))'
ENTRYPOINT R -e 'runApp()'
```
After `forked.py` runs parallel simulations that explore different synaptic weight values, the file `sa.py` (spike analysis) is called. `sa.py` is and analysis program which finds firing rates of cells, ISIs, Coefficients of Variation, Spike Distance Matrices, and many other network level feature analysis.

Spike analysis is called using `dask distributed's` parallel map function, to do a parallel analysis that acts membrane and spike time recordings generated by the previously described simulations launched by `forked.py`.

The graphs generated by `sa.py` (spike analysis) file where then loaded into a dedicated notebook `RichClubPresentation.ipynb` with code for interactively stepping through different values of synaptic weight, in a way that enabled us to interrogate the contribution of synaptic weight values on network dynamics.  

To outline the entire workflow: We created a Dedicated Docker container, that had preinstalled `PyNN`, `NEURON`, `elephant`, `neo` and `R`. Inside docker container we ran the files: 'forked.py', 'sa.py'. We also anticipate the near future possibility of running: ```RUN R -e 'install.packages(c("rPython","shiny","igraph","visNetwork,"pracma,"stringr","chorddiag"))'
ENTRYPOINT R -e 'runApp()```

Daniel Petty's final network graph presents structure and function relationships in a manner that is readily digested by the human visual system. Firing rate, neuron centrality, projection partners and neuron type are all present on the same graph.
3D geometry of the neuronal network and exact spike timing data, have been thrown away in favor of these more abstract and predictive relationships. Daniel's code exposes structure function relationships which would otherwise be buried in matricies, making them much more accessible to the human network modeller.


_[1]    C. L. Rees, D. W. Wheeler, D. J. Hamilton, C. M. White, A. O. Komendantov, and G. A. Ascoli, “Graph theoretic and motif analyses of the hippocampal neuron type potential connectome,” Eneuro, vol. 3, no. 6, p. ENEURO–0205, 2016._


_[2]    D. Qi and Z. Xiao, “Spike trains synchrony with different coupling strengths in a hippocampus CA3 small-world network model,” in Biomedical Engineering and Informatics (BMEI), 2013 6th International Conference on, 2013, pp. 270–275."_








