<h1 class="code-line" data-line-start=0 data-line-end=1 ><a id="Scattered_data_interpolation_0"></a>Scattered data interpolation</h1>
<p class="has-line-data" data-line-start="1" data-line-end="5">Many tasks start with gathering the data by scanning the real world. Often, this data comes in an<br>
unstructured (scattered) form, where each sample may fall anywhere in the domain. This so-called<br>
scattered data must not be confused with unstructured data, where the samples themselves have no<br>
specified structure.</p>
<p class="has-line-data" data-line-start="6" data-line-end="9">The scattered data represents just a small set of values of a spatially varying quantity, which we often<br>
want to evaluate not only at the sample locations but also at arbitrary locations between the samples.<br>
We therefore need to use some kind of interpolation to be able to get a full view of the data.</p>
<h2 class="code-line" data-line-start=10 data-line-end=11 ><a id="Program_10"></a>Program</h2>
<p class="has-line-data" data-line-start="12" data-line-end="13">Program implements both the basic and the modified Shepard’s method in order to interpolate the input samples and output a regular grid (a volume) that can be visualized in a volume rendering application. The program will be given a bounding box in order to construct the volume, and the parameters the interpolation itself. The distance metric is be Euclidean. To accelerate the search  the nearest neighbors and octree are used.</p>
