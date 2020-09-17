# processing-household-composition-data

The scripts in this repository construct histograms of age-stratified household compositions from census data, primarily for use in infectious disease modelling. There are a few key pieces of terminology we will use in this readme. An *age class* is the set of people in a population who are in a specific age range. A *household* is a group of indivduals who are grouped together for epidemiological purposes, typically because they live together and/or form a nuclear family. A *household composition* is a vector indexed by the set of age classes which specifies the number of individuals in each age class present in a household. The composition of a household is defined up to the age classes chosen, so that the set of compositions present in a population are dependent on the age classes being used. The functions provided in this repository are designed to produce histograms of household compositions based on user-specified age classes, and to any of the spatial resolutions for which raw data exists. A major restriction is that we only allow for age classes which are a *coarsening* of the age classes used in the raw data, in the sense that each of the user-specified age classes must be either one of the age classes in the raw data or a union of consecutive age classes. For instance, if the raw data has age classes (0-20,20-40,40-60,60-80,80+), then (0-40.40-60.60+) is allowed but (0-35,35-45.45-60,60+) is not.

These scripts are primarily designed for processing the household data which is available from the UK's Office for National Statistics (ONS). The `preprocessing` subfolder is for scripts which reformat specific datasets to make them compatible with the rest of the code in the repository.

## Glossary

### CT1088
This is a dataset compiled by the ONS, containing ten-year age band composition data for all households of size six or fewer in England and Wales. Each data point consists of a spatial location, a household composition in terms of the number of people in each age class in a household, and the number of households of that composition in that location. Because of the large size of this dataset, it is separated into twelve multi-sheet Excel spreadsheets coresponding to different regions of England, and one for Wales.

### CT1089
This is the equivalent of the CT1088 dataset for households of size seven or more. The age bands in this dataset are 0-19 years, 20-69 years, and 70+ years. The composition data for all the output areas in England and Wales are stored in a single sheet.

### Output areas (OAs) and Super Output Areas (SOAs)

An output area is a spatial units used by the ONS. For our purposes, a single output area consists of all the households in a given set of postcodes. A Super Output Area (SOA) is an aggregation of output areas. There are different levels of SOA, with each one consisting of an aggregation of SOAs at the level below. In ascending size order, the different levels of SOA are lower layer super output area (LSOA), middle layer super output area (MSOA), local authority (LA), and country. For more information, see https://www.ons.gov.uk/census/2001censusandearlier/dataandproducts/outputgeography/outputareas and https://www.ons.gov.uk/methodology/geography/ukgeographies/censusgeography#super-output-area-soa.

## preprocessing

The scripts in this folder are designed to gather and prepare household compostion data for use in the rest of the repository. Before running any of the examples using England-and-Wales data, you will need to first download and reformat this data. `download_CT1088_data` downloads the England-and-Wales household composition data available from the ONS (available manually from https://www.ons.gov.uk/searchdata?q=CT1088*&sortBy=relevance&q=CT1088* and https://www.ons.gov.uk/peoplepopulationandcommunity/housing/adhocs/11569ct10892011census) and saves it to a subfolder `data/CT1088_tables`. This script requires an internet connection. The CT1089 dataset, containing household composition data for households of size seven or more, is available as a single .xlsx file, whereas the CT1088 dataset, containing household composition data for households of size six or less, is split across thirteen .xlsx files, each containing multiple sheets. The script `combine_CT1088_tables` replaces the multi-sheet .xlsx files with .csv files which can be handled more efficiently by the other scripts in the repository. `combine_CT1088_tables_par` carries out the same operations using parallel computing, and should therefore be substantially faster than `combine_CT1088_tables`. `combine_CT1088_tables_par` is set up to use four threads. Without parallelisation, execution takes 226 minutes and 10 seconds; with four threads, this is reduced to 90 minutes and 8 seconds.

To download the ONS household composition data and prepare it for use, run either
```
download_CT1088_data
clear
combine_CT1088_tables
clear
```
or
```
download_CT1088_data
clear
combine_CT1088_tables_par
clear
```


## functions

### `load_CT1088.m`

This function loads the CT1088 tables into a single Matlab table. This contains household composition data for all the households of size six or fewer in the UK, to the finest spatial resolution available. It is not recommended that users attempt to save the combined table as a single .csv file, as its large size means writing to file will take an extremely long time.

### `filter_rare_households_ONS.m`

This function is used to remove the largest households from a datase. It takes a household composition dataset in ONS format and a number *p* as arguments, and constructs a weighted histogram of household sizes, such that the size of the *n*th bin in the histogram is the proportion of the population belonging to households of size $n$ (**not** the proportion of households which are size *n*). Using this histogram, we calculate the proportion of the population in households of size *n* or higher, for *n* ranging from 1 to the maximum household size found in the composition data. From these proportions we choose a size threshold *N* such that *N* is the smallest possible household size for which the proportion of the population in households of size *N* or larger is less than *p*. All households of size *N* or larger are removed from the household composition data, leaving us with a version of the original composition data which has been filtered to minimise the maximum household size while keeping the proportion of the population removed from the data below *p*. For instance, if we set *p=0.05* and find that 7.5\% of the population belong to households of size 6 or larger and 2.5% belong to households of size 7 or larger, then `filter_rare_households_ONS` will remove all households of size 7 or larger, because removing the households of size 6 will mean we have removed more than 5% of the population, while on the other hand keeping the households of size 7 is not necessary in light of the proportion constraint. The motivation for removing large households is that the number of epidemiological states a household can occupy increases dramatically with the household size and the number of age classes present, meaning that including large, rare household compositions in a model population substantially increases system size and computational intensity in exchange for a relatively small payoff in terms of increased accuracy.

### `build_HH_dist_from_ONS_data.m`

This function constructs a distribution or set of distributions of household compositions at a specified spatial resolution based on a household composition dataset in ONS format. The ONS data specifies the compositions present in each of a given low-level output area (ENUMOA in the CT1088 data) and the frequencies at which they appear, and for each such datapoint it specifies the higher-level output areas to which the low-level output area belongs (the different level output areas are nested so that each lower level output area belongs to exactly one output area at the next lowest level). In practice, we would like to know the compositions present, and their frequencies, in higher-level output areas; for instance, we may wish to model infectious disease dynamics for the whole of England and Wales rather than any particular region in either country. This requires aggregating all of the household compositions and frequencies for all the lowest-level output areas contained in each of the output areas at the resolution we are interested in; to model the whole of England and Wales we would need to aggregate all of the data into a single non-repeating list of compositions, while to model each of England and Wales we would need to aggregate the compositions and frequencies from the lowest-level output areas in England into one histogram and those from Wales into another. This turns out to be an extremely computationally intensive task because to calculate the number of times a given frequency appears in a given higher-level output area we need to search for all of its appearances in the lowest-level output areas lying within that higher-level output area. This is a computationally intensive task because under a naive approach a comparison between two compositions will involve a comparison between the numbers of individuals in each age class in the two compositions. If the composition dataset is in terms of *K* age classes, then each comparison operation will involve up to *K* comparisons. Because of the large size of low-level output area household composition datasets (the CT1088 dataset contains over eight million datapoints) it is important to perform any operations on the data as efficiently as possible. To this end, `build_HH_dist_from_ONS_data` proceeds by encoding each composition as a unique integer, meaning each comparison is reduced to a single operation. With the compositions encoded as integers, we can replace the columns corresponding to the compositions in the original dataset with a single composition containing the appropriate integers. For each output area we then construct a histogram of the integers which appear in rows where the appropriate output area code appears, and obtain a distribution of household compositions for each output area by replacing the integers with the compositions they encode. Note that in the code as it appears here, we find the rows corresponding to a given output area using a string comparison, although it would be more efficient to also encode these strings as integers, and we will do this in a future update to the repository.

The encoding we use is as follows. Let *N_k* be the number of individuals of age class *k* present in a given household composition, and let $M_k$ be the maximum number of individuals of age class *k* which is observed in a single household across the entire population. We encode each household according to the formula <img src="https://render.githubusercontent.com/render/math?math=N_1%2B\sum_k N_k\prod_{j=1}^{k-1}(1%2BM_j)">. To see that this formula does indeed uniquely encode the compositions, compare with the representation of the integers in base ten, where, for instance, the number 1,234 is encoded as <img src="https://render.githubusercontent.com/render/math?math=1\times10^3%2B2\times10^2%2B3\times10^1%2B4\times10^0">. For the purposes of this analogy, it is important to emphasise that the powers of ten represent successive multiplications by the number of possible values the digit in the following places can take, i.e. a 3 in the tens column represents three successive cycles over all the values that can go in the units column. Our encoding of the household compositions does a very similar thing, but the maximum number of individuals of each age class replaces the factors of 10, because now the number of values to cycle over in each column is given by the maximum value observed in the population. While this is not the most efficient possible encoding (where a more efficient encoding means one that produces smaller integers, which is desirable because very large numbers can not be stored as 64-bit integers), it has the advantage of being easy to explain and implement in code.

### `calculate_system_size.m`

Once a household composition distribution has been constructed, it is useful to know how large a household-structured infectious disease model based on that distribution is, in terms of the number of states in the Markov chain defined by the model. This allows the user to assess the computational intensity of the resulting model before saving a household composition distribution. By using a few different thresholds in `filter_rare_households_ONS` and applying `calculate_system_size` to the composition distributions which follow from these thresholds, one can find an optimal balance between the amount of household data retained in the model population and the computational power to implement the model. `calculate_system_size` calculates the number of states in the Markov chain defined by a set of household compositions (the distribution across these states will not affect the number of the states in the chain but will clearly contribute to its behaviour) and a given number of compartments. It does this using the [stars and bars](https://en.wikipedia.org/wiki/Stars_and_bars_(combinatorics)) formula from combinatorics, which calculates the number of ways we can assign *n* balls to *k* compartments; for each age class in a household, this means assigning each of the individuals in the age class to one of the model compartments. The number of states a household of a given composition can occupy is the product of the stars and bars formulae for all the age classes present in that household, and then the number of states the entire population can occupy is the sum of the household-level system sizes for all the compositions present in the population.

We stress that the system size calcualted by this function is that of a mean-field model of an infinite number of households, where the quantities simulated are the proportions of households in each composition and epidemiological state. The system size of a model of a finite number of households (for instance a network model) will be the product of the household-level system sizes, rather than the sum, with each household composition repeated in the product for every time it appears in the model population.

## preprocessing

## examples

### `generate_england_wales_adult_child_population.m`

This example demonstrates how to construct a single household composition distribution for the whole of England and Wales from the CT1088 and CT1089 data. We start by loading the two datasets and recasting them in terms of two age classes, one consisting of children (0-19 years old) and one consisting of adults (20+ years old). We do this by converting the composition columns of the household composition data tables to arrays, and then post-multiplying by a matrix which adds up all the entries in the "child" age classes and all the entries in the "adult" age classes in each row and create a new two-column array with each row consisting of these two sums. If we have *K* age classes and wish to merge these into *L* age classes then the required matrix will have dimensions *K* by *L* and will have element *(k,l)* equal to 1 if age class *k* is contained within age class *l* and 0 otherwise. Once we have recast the data in terms of two age classes, we combine the CT1088 and CT1089 data into a single table summarising the entirety of England and Wales, and apply `filter_rare_households_ONS` with a threshold of 0.05, which removes all households of size seven and higher (coincidentally, these are precisely the households in the CT1089 data). This filtered household data is then inputted to `build_HH_dist_from_ONS_data` with the resolution set to `ALL`, to create an England-and-Wales-level household composition distribution. We then apply `calculate_system_size` for a few illustrative numbers of compartments.

### `generate_england_wales_adult_child_vulnerable_population.m`