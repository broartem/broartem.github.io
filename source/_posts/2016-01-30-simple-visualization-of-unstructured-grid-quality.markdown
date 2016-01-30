---
title: Simple visualization of unstructured grid quality with Python and VTK
layout: post
---

The Visualization ToolKit (VTK) is an excellent open source library for creating visualization applications. VTK has been written in C++ and has an automatically generated Python wrappings around the majority of its C++ classes. This post shows how to build a simple [mesh quality](http://www.vtk.org/Wiki/VTK/mesh_quality) visualization of unstructured tetrahedral grid from Python using VTK. The result should look like the one on the picture below:

![mk8-gear-threaded-cell-quality.png]({{ "/images/" | prepend: site.baseurl }}/2016-01/mk8-gear-threaded-cell-quality.png)

The color of each individual tetrahedron on the picture corresponds to its Aspect Ratio value. This correspondence is reflected on the legend. In general, mesh quality is a positive real number which relates to the shape of a particular cell in a mesh. Quality metrics play an important role in finite element analysis, because the shape of individual cells can significantly affect the accuracy of a simulation. Internally, VTK uses [VERDICT](https://cubit.sandia.gov/public/verdict.html) suite of mesh quality metrics for cell processing. For more information on this topic refer to the [VTK/mesh quality](http://www.vtk.org/Wiki/VTK/mesh_quality) official wiki page and [Verdict Manual, rev. A](http://www.vtk.org/Wiki/images/6/6b/VerdictManual-revA.pdf) (3 MB)

## Prerequisites
I assume that you have a basic familiarity with Python. Also, you have to set up a [Python environment](http://www.vtk.org/Wiki/VTK/Tutorials/PythonEnvironmentSetup) for VTK development. Personally, I have found an article about VTK on [The Architecture of Open Source Applications](http://aosabook.org/en/vtk.html) site to be the best intro to VTK and its philosophy.

## Input dataset
I used MK8 Filament Drive Gear STEP file from [here](https://grabcad.com/library/mk8-filament-drive-gear-1) to generate an unstructured VTK mesh using [Gmsh](http://geuz.org/gmsh/) finite element mesh generator. You may just grab generated VTK file [mk8-gear-threaded.vtk](https://raw.githubusercontent.com/broartem/various-datasets/master/data/geometry/mk8-gear-threaded.vtk) from my dataset repository.

## Building blocks
In this section we are going to define three basic functions to read an unstructured grid from a *.vtk file, to apply a mesh quality filter to it and to generate a custom lookup table. In the subsequent section we will create a [visualization pipeline](http://www.infovis-wiki.net/index.php/Visualization_Pipeline) using these functions as building blocks.

The first function just reads an unstructured grid from a *.vtk file using [vtkUnstructuredGridReader](http://www.vtk.org/doc/nightly/html/classvtkUnstructuredGridReader.html) and returns an instance of [vtkUnstructuredGrid](http://www.vtk.org/doc/nightly/html/classvtkUnstructuredGrid.html) class.

{% highlight python %}
def read_unstructured_grid(filepath):
    """
    Reads unstructured grid from a VTK file
    :param filepath: Path to *.vtk file, containing unstructured grid
    :return: instanse of vtkUnstructuredGrid
    """
    reader = vtk.vtkUnstructuredGridReader()
    reader.SetFileName(filepath)
    reader.Update()
    return reader.GetOutput()
{% endhighlight %}

The second function applies [vtkMeshQuality](http://www.vtk.org/doc/nightly/html/classvtkMeshQuality.html) filter to a grid of type vtkUnstructuredGrid with a specific tetrahedron quality measure defined among vtk.VTK_QUALITY_* constants. Also, vtkMeshQuality class defines functions     SetTriangleQualityMeasure, SetQuadQualityMeasure, SetHexQualityMeasure for various types of cells. So, you are not restricted by tetrahedral grids only and can visualize even hybrid grids.

{% highlight python %}
def apply_quality(grid, measure):
    """
    Applies vtkMeshQuality filter to an input grid
    :param grid: input grid, instance of vtkUnstructuredGrid
    :param measure: one of VTK constants vtk.VTK_QUALITY_*
    :return: instance of vtkMeshQuality
    """
    quality = vtk.vtkMeshQuality()
    quality.SetInputData(grid)
    quality.SetTetQualityMeasure(measure)
    quality.Update()
    return quality
{% endhighlight %}

The third function yields a lookup table object of type vtkLogLookupTable using custom diverging color map scheme based on Kenneth Moreland's article [Diverging Color Maps for Scientific Visualization](http://www.kennethmoreland.com/color-maps). The article points out three major reasons to avoid widely used rainbow color map:

* The colors do not follow any natural perceived ordering
* The perceptual changes in the colors are not uniform
* It is sensitive to deficiencies in vision. Roughly 5% of the population cannot distinguish between the red and green colors

In general, the color map should be tuned for any particular visualization. But if you do not have enouth time, or experience, or incentive to do it, just consider using any default general purpose diverging color map among those suggested in the article. That is exactly what I did in the code snippet below.

{% highlight python %}
def generate_lookup_table():
    """
    Generate a lookup table based on diverging color map schema from
    RBG(0.230, 0.299,  0.754) to RGB(0.706, 0.016, 0.150)
    see http://www.kennethmoreland.com/color-maps/
    :return: vtkLogLookupTable
    """
    lookup_table = vtk.vtkLogLookupTable()
    num_colors = 256
    lookup_table.SetNumberOfTableValues(num_colors)

    transfer_func = vtk.vtkColorTransferFunction()
    transfer_func.SetColorSpaceToDiverging()
    transfer_func.AddRGBPoint(0, 0.230, 0.299,  0.754)
    transfer_func.AddRGBPoint(1, 0.706, 0.016, 0.150)

    for ii,ss in enumerate([float(xx)/float(num_colors) for xx in range(num_colors)]):
        cc = transfer_func.GetColor(ss)
        lookup_table.SetTableValue(ii, cc[0], cc[1], cc[2], 1.0)

    return lookup_table
{% endhighlight %}

Compare the rainbow color map (left) used by vtkLogLookupTable by default to our custom diverging one (right) on the image below. It is easier to percieve a diverging scheme.

![mk8-gear-threaded-color-map-comparison.png]({{ "/images/" | prepend: site.baseurl }}/2016-01/mk8-gear-threaded-color-map-comparison.png)

But why do we use a log scale lookup table? The reason lies in the fact that if we plot an [order statistics](https://en.wikipedia.org/wiki/Order_statistic) for a grid's cell quality array (the k'th order statistics is equal to the k'th smallest quality value in our grid) it is likely that starting from some point the plot will become pretty steep. And majority of cells in our grid will have colors occupying a tiny fraction of the whole color range. So, we use a log scale lookup table to fix this bias toward poor quality cells. Depending on your use case you may need this or not. If you don't, just use vtkLookupTable class instead of vtkLogLookupTable. The order statistics plot for our mesh looks like the following:

![mk8-gear-quality-plot.png]({{ "/images/" | prepend: site.baseurl }}/2016-01/mk8-gear-quality-plot.png)

On the picture below you can see the difference between the vtkLookupTable (left) and vtkLogLookupTable (right). The image on the right show more contrast than the one on the left.

![mk8-gear-log-vs-linear-lookup-table.png]({{ "/images/" | prepend: site.baseurl }}/2016-01/mk8-gear-log-vs-linear-lookup-table.png)

## Visualization pipeline
It is time to combine our building blocks in order to produce a visualization pipeline.

Read input file:
{% highlight python %}
filepath = "/path/to/mk8-gear-threaded.vtk"
grid = read_unstructured_grid(filepath)
{% endhighlight %}

Apply mesh quality filter with quality metric of Aspect Ratio:
{% highlight python %}
quality = apply_quality(grid, vtk.VTK_QUALITY_ASPECT_RATIO)
{% endhighlight %}

Create a mapper. Generally speaking, mapper maps data to graphics primitives (see [vtkMapper](http://www.vtk.org/doc/nightly/html/classvtkMapper.html) interface description for details):
{% highlight python %}
mapper = vtk.vtkDataSetMapper()
mapper.SetInputData(quality.GetOutput())

quality_min = quality.GetOutput().GetFieldData()\
    .GetArray('Mesh Tetrahedron Quality').GetComponent(0, 0)
quality_max = quality.GetOutput().GetFieldData()\
    .GetArray('Mesh Tetrahedron Quality').GetComponent(0, 2)

lookuptable = generate_lookup_table()
lookuptable.SetTableRange(quality_min, quality_max)

mapper.SetScalarRange(quality_min, quality_max)
mapper.SetLookupTable(lookuptable)
{% endhighlight %}

Create a grid actor. Actor represent an entity in a rendering scene (entity position, scaling, and so on, see [vtkActor](http://www.vtk.org/doc/nightly/html/classvtkActor.html)):
{% highlight python %}
actor = vtk.vtkActor()
actor.SetMapper(mapper)
actor.GetProperty().EdgeVisibilityOn()
{% endhighlight %}

Create a legend actor:
{% highlight python %}
legend_actor = vtk.vtkScalarBarActor()
legend_actor.SetLookupTable(lookuptable)
{% endhighlight %}

Initialize a camera position. Actually, we don't have to set up a camera object because VTK will do this for us by default. The Position property controls the camera's position in 3D space. The FocalPoint property controls the point the camera "looks" at and the ViewUp specifies the
direction that goes bottom-to-top in the view:
{% highlight python %}
camera = vtk.vtkCamera()
camera.SetPosition(-27, 300, 170)
camera.SetFocalPoint(0, 0, 50)
camera.SetViewUp(0, -0.5, 1)
{% endhighlight %}

Create a renderer:
{% highlight python %}
renderer = vtk.vtkRenderer()
renderer.AddActor(actor)
renderer.AddActor(legend_actor)
renderer.SetBackground(1, 1, 1)
renderer.SetActiveCamera(camera)
{% endhighlight %}

Create a renderer window and show the result:
{% highlight python %}
renderer_window = vtk.vtkRenderWindow()
renderer_window.AddRenderer(renderer)

interactor = vtk.vtkRenderWindowInteractor()
interactor.SetRenderWindow(renderer_window)

interactor_style = vtk.vtkInteractorStyleTrackballCamera()  # ParaView-like interaction
interactor.SetInteractorStyle(interactor_style)

interactor.Initialize()
interactor.Start()
{% endhighlight %}

Full source code:

{% gist 35915253da03424bd1c9 %}