# TOPMed_merge
1. Required packages and versions
    1. This project is built with python 3.7.x
    2. Packages that might need manual installation: pandas, bgzip
        1. To install missing packages use: ```pip install package_name```
        2. For example: ```pip install bgzip```

2. Arguments (flags)
    - The whole program will be used in command line with user input flags and values
    - Valid flags are:
        - --input: (required) files to be merged, multiple files are allowed
        - --output: (optional, default is "merged") output file name without suffix. Default value is "merged" and saved at working directory (plus appropriate suffix).
        - --thread (optional) Defines how many threads to use in multiprocessing. Default value is
            - Valid values are integers. If number of threads > number of cpus of the system, will use number of cpus instead of user supplied value. If number of threads <0, will use 1 instead of user supplied value.
        - --missing: (optional, default is 0) Defines number of missing values allowed for each variant. Cannot exceed total number of files to be merged. If a variant is missing for some individuals, the values will be ".|." (or other user supplied value with --na_rep as "na_rep|na_rep") in merged output file. If --missing is 0, only variants shared by all input files will be saved in merged result.
        - --na_rep: (optional) Defines what symbol to use for missing values. Default is ".". This flag is ignored if --missing is 0.
        - --r2_threshold: (optional, default is 0, ie. no filtering) Only variants with combined imputation quality r2_combined≥r2_threshold will be saved in the merged file
        - --r2_output: (optional, default is "first") defines how r2 is calculated in the output file. Valid values are:
            - first: output r2 from the first file. In order to use this setting and avoid missing r2, "--missing" must be 0.
            - weighted_average: calculated weighted average of r2. Weight is determined by number of individuals of each file.
            - mean, min, max: Mean, min or max of r2, ignore missing values
        - --duplicate_id: (optional, default is 0) Defines number of duplicated individuals in each input file. Duplicated IDs should be the first N columns in each file and not mixed with unique IDs.
            - For example (in our case), the first 100 individuals in each input file are duplicated as a sanity check. Set --duplicated_id to 100 so that only one set of these IDs will be saved in output file.
            - (ie. starting from the second input file, data of the first 100 individuals will be skipped in the merged output)
        - --help: (optional)  Exit program after printing out help info, ignore any other flags and values provided.
3. Requirements and special notes:
    1. All input files should be bgzipped, post-imputation VCF files from TOMed, with name as [file_name].vcf.gz
    2. Input vcf.gz files should be sorted by position (looks like TOPMed output is sorted by default, so this should not be not a problem)
    3. Corresponding gzipped info files should be stored in the same directory as [file_name].info.gz
    4. All input dose.vcf.gz files should have the same number of header lines (ie. lines start with "##")
    5. Do not move or modify variants_excluded.txt and variants_kept.txt until current run is finished.
    6. Output files will be overwritten if another run saves output in the same directory with the same file names.
    7. Value smaller than 0.000001 (6 digits of precision) will be rounded to 0 when outputting ALT_frq, MAF and R2 into variant_kept.txt and variant_excluded.txt. These values will also be used to replace INFO column in merged dose.vcf.gz file. Precision can be changed with float_format parameter in get_SNP_list.py.
4. Calculation of combined r2 and MAF
    1. r2
        1.1 Mean:
	    $$r^2_{combined} = \frac{\sum_{i=1}^{n} r^2_{i}}{n}$$
            - $r^2_i$: Imputation quality $r^2$ of the $i$th input file
            - $n$: Total number of input files to be merged
	    - ignore missing values in calculation
        1.2 Weighted average, **ignore NAs**
            $$r^2_{combined} = \frac{\sum_{i=1}^{n}r_i^2 * N_i}{\sum_{i=1}^{n}N_i}$$
            - $r^2_i$: Imputation quality $r^2$ of the $i$th input file
            - $n$: Total number of input files to be merged
            - Ignore missing values. For example a variant has below r2 in each input file:
                - File #1: r2=0.3, number of individuals = 1000
                - File #2: Missing, number of individuals = 2000 (← Ignore this)
                - File #3: r2=0.2, number of individuals = 3000
                - Weight MAF = (0.3*1000 + 0.2*3000)/(1000 + 3000) = 0.225
    2. MAF: weighted average, **include NAs:** Use the same equation as r2
        $$MAF_{combined} = \frac{\sum_{i=1}^{n}MAF_i * N_i}{\sum_{i=1}^{n}N_i}$$ 
        - $r^2_i$: Imputation quality $r^2$ of the $i$th input file
        - $n$: Total number of input files to be merged
        - **Include missing values** when calculate total number of individuals. For example, a variant has below frequencies in each input file:
            - File #1: MAF=0.2, number of individuals = 1000
            - File #2: Missing, number of individuals = 2000
            - File #3: MAF=0.1, number of individuals = 3000
            - Weight MAF = (0.2*1000 + 0.1*3000)/(1000 + 2000 + 3000) = 0.083
5. Example code using sample data in ./data_sample/, output files saved in ./output_sample/
```
python merge_files.py --input ../data_sample/sample_group1.dose.vcf.gz ../data_sample/sample_group2.dose.vcf.gz ../data_sample/sample_group3.dose.vcf.gz \
		      --output ../output_sample/merged_sample \
		      --missing 1 \
		      --duplicate_id 5 \
		      --r2_output weighted_average
```
