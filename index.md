# `cell_cycle_time_course`

`cell_cycle_time_course` is a repository that stores time courses of cell cycle regulators.

{% assign doclist = site.pages | sort: 'url'  %}
<ul>
   {% for doc in doclist %}
        {% if doc.name contains '.md' or doc.name contains '.html' %}
            <li><a href="{{ site.baseurl }}{{ doc.url }}">{{ doc.url }}</a></li>
        {% endif %}
    {% endfor %}
</ul>

## Table of Contents
* 4i_dent_lang
    * 2019082X_RPE1
    * 2019XXXX_RPE1_PCNA-fluorescent_protein
        * MK1775_treated_sorted
            * C10
                * curation
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd1_0_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd3_2_A01_C01.html
                    * DNA_vs_CCNA.html
                    * Mean_Morphology_Area.html
                    * Nuclei_Mean_Morphology_Area.html
                    * RP1_p_vs_CCNA.html
                    * detached_metric.html
                    * segmentation_metric.html
                    * std_dapi_nucleus.html
                * kalman.html
                * parzen.html
                * raw.html
            * E10
                * curation
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd1_0_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd3_2_A01_C01.html
                    * DNA_vs_CCNA.html
                    * Mean_Morphology_Area.html
                    * Nuclei_Mean_Morphology_Area.html
                    * RP1_p_vs_CCNA.html
                    * detached_metric.html
                    * segmentation_metric.html
                    * std_dapi_nucleus.html
                * kalman.html
                * parzen.html
                * raw.html
            * G10
                * curation
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd1_0_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd3_2_A01_C01.html
                    * DNA_vs_CCNA.html
                    * Mean_Morphology_Area.html
                    * Nuclei_Mean_Morphology_Area.html
                    * RP1_p_vs_CCNA.html
                    * detached_metric.html
                    * segmentation_metric.html
                    * std_dapi_nucleus.html
                * kalman.html
                * parzen.html
                * raw.html
        * untreated_sorted
            * C09
                * curation
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd1_0_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd3_2_A01_C01.html
                    * DNA_vs_CCNA.html
                    * Mean_Morphology_Area.html
                    * Nuclei_Mean_Morphology_Area.html
                    * RP1_p_vs_CCNA.html
                    * detached_metric.html
                    * segmentation_metric.html
                    * std_dapi_nucleus.html
                * kalman.html
                * parzen.html
                * raw.html
            * E09
                * curation
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd1_0_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd3_2_A01_C01.html
                    * DNA_vs_CCNA.html
                    * Mean_Morphology_Area.html
                    * Nuclei_Mean_Morphology_Area.html
                    * RP1_p_vs_CCNA.html
                    * detached_metric.html
                    * segmentation_metric.html
                    * std_dapi_nucleus.html
                * kalman.html
                * parzen.html
                * raw.html
            * G09
                * curation
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd1_0_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01.html
                    * Cytoplasm_Mean_Intensity_mean_DAPI_Rd3_2_A01_C01.html
                    * DNA_vs_CCNA.html
                    * Mean_Morphology_Area.html
                    * Nuclei_Mean_Morphology_Area.html
                    * RP1_p_vs_CCNA.html
                    * detached_metric.html
                    * segmentation_metric.html
                    * std_dapi_nucleus.html
                * kalman.html
                * parzen.html
                * raw.html

* 4i_stallaert

## Description

### `lit_review`
* Raw data and aggregated `time_course_data.xlsx` from proteomic datasets of [Ly et al (2014)](https://doi.org/10.7554/eLife.01630), [Ly et al 2017](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5650473/) and [Mahdessian et al 2019](https://www.biorxiv.org/content/10.1101/543231v1) and a transcriptomic dataset of [Leng et al (2015)](https://www.nature.com/articles/nmeth.3549). Data is rescaled to match guesstimates of average concentration ([Yin Hoon Chew and Jonathan Karr](https://github.com/KarrLab/h1_hesc/blob/master/h1_hesc/kb_gen/core.xlsx)) in units of mol/l.

### `4i_dent_lang`
* Time courses of WEE1 inhibited and untreated cells from own experiments, reconstructed with reCAT.

### `4i_stallaert`
* Time courses of untreated cells from [Stallaert et al](https://www.biorxiv.org/content/10.1101/2021.02.11.430845v1), reconstructed with reCAT.

## Questions and comments
Please contact [Paul F. Lang](mailto:paul.lang@wolfson.ox.ac.uk) with any questions or comments.
