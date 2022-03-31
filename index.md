# `cell_cycle_time_course`

`cell_cycle_time_course` is a repository that stores time courses of cell cycle regulators. Unfortunately, GitHub does not render interactive figures in HTML files. For this reason, we are hosting all HTML files on the `cell_cycle_time_course` repository on this website. You can access individual HTML files by suffixing their path to `https://paulflang.github.io/cell_cycle_time_course/`. For instance, the file `4i_stallaert/2021_Stallaert_cycling_reCAT/kalman.html` can be accessed through [https://paulflang.github.io/cell_cycle_time_course/4i_stallaert/2021_Stallaert_cycling_reCAT/kalman.html](https://paulflang.github.io/cell_cycle_time_course/4i_stallaert/2021_Stallaert_cycling_reCAT/kalman.html) (allow a few seconds to load). An exhaustive list of all HTML files in this repository is listed below. Alternatively, the HTML files in the GitHub repository can be downloaded and opened with a web browser of your choice.

## Description

### `lit_review`
* Does not contain HTML files, therefore only available on the GitHub repository.

### `4i_dent_lang`
* Time courses of WEE1 inhibited and untreated cells from own experiments, reconstructed with reCAT.

### `4i_stallaert`
* Time courses of untreated cells from [Stallaert et al](https://www.biorxiv.org/content/10.1101/2021.02.11.430845v1), reconstructed with reCAT.

## Questions and comments
Please contact [Paul F. Lang](mailto:paul.lang@wolfson.ox.ac.uk) with any questions or comments.

## Content list

{% assign doclist = site.pages | sort: 'url'  %}
<ul>
   {% for doc in doclist %}
        {% if doc.name contains '.md' or doc.name contains '.html' %}
            <li><a href="{{ site.baseurl }}{{ doc.url }}">{{ doc.url }}</a></li>
        {% endif %}
    {% endfor %}
</ul>

