
=========================
CBCT module documentation
=========================

Overview
--------

.. automodule:: pylinac.cbct

Running the Demo
----------------

To run the CBCT demo, create a script or start in interpreter and input::

    from pylinac import CBCT
    CBCT().run_demo() # the demo is a Varian high quality head scan

Results will be printed to the console and a figure showing the slices analyzed will pop up::

    - CBCT QA Test -
    HU Regions:  {'PMP': -200.34063745019921, 'Acrylic': 116.68700787401575, 'Poly': -45.015748031496067, 'Teflon': 997.26494023904388, 'Air': -997.61220472440948, 'LDPE': -103.43503937007874, 'Delrin': 341.14763779527561}
    HU Passed?:  True
    Uniformity:  {'Top': 2.9074662430500395, 'Bottom': 5.9872915011914216, 'Right': -0.35345512311358218, 'Left': 10.128673550436854, 'Center': 21.321286735504369}
    Uniformity Passed?:  True
    MTF 80% (lp/mm):  1.09
    Geometric distances:  {'Top-Horiz': 49.92540443205539, 'Bottom-Horiz': 49.965520922765805, 'Right-Vert': 49.91174913443049, 'Left-Vert': 50.078027293396}
    Geometry Passed?:  True

.. plot:: pyplots/cbct_demo.py

Typical Use
-----------

CBCT analysis as done by this module closely follows what is specified in the CatPhan manuals, replacing the need for hand measurements.
Assuming you've made a CBCT object as follows::

    from pylinac.cbct import CBCT
    mycbct = CBCT()

The minimum needed to get going is to:

* **Load images** -- Loading the DICOM images into your CBCT object can be done by passing the folder the images are located in.
  This can be done directly, or by using a UI. The code might look like either of the following::

    # set the folder path
    cbct_folder = r"C:/QA Folder/CBCT/June monthly"  # use of 'r' is for raw string; otherwise spaces and backslashes aren't interpreted properly
    # load the images from the file path
    mycbct.load_folder(cbct_folder)

    # *OR*

    zip_file = r"C:/QA Folder/CBCT/June monthly.zip"
    mycbct.load_zip_file(zip_file)

    # *OR*

    # Identify the folder using a UI
    mycbct.load_folder_UI()


* **Analyze the images** -- Once the folder/images are loaded, tell CBCT to start analyzing the images. See the
  Algorithm section for details on how this is done::

    mycbct.analyze()

* **View the results** -- The CBCT module can print out the summary of results to the console as well as draw a matplotlib image to show where the
  samples were taken and their values::

      # print results to the console
      mycbct.return_results()
      # view analyzed images
      mycbct.plot_analyzed_image()

Algorithm
---------

The CBCT module is based on the tests and values given in the CatPhan 504 Manual. The algorithm works like such:

**Allowances**

* The images can be any size.
* For Varian machines, the images can be acquired with any protocol (Pelvis, Head, etc).
* The phantom can have significant translation in the Left-Right, Up-Down direction; i.e. the setup does not have
  to be precise.
* The phantom can have significant roll and moderate yaw and pitch.

**Restrictions**

    .. warning:: Analysis can catastrophically fail or give unreliable results if any Restriction is violated.

* The phantom used must be an unmodified CatPhan 504, as endorsed and supplied by Varian.
* The phantom must have <0.5cm offset in the z (In-Out) direction (work to remove this is in the plans).



**Pre-Analysis**

* **Determine image properties** -- Upon load, the image set is analyzed for its DICOM properties to determine mm/pixel
  spacing, rescale intercept and slope, manufacturer, etc. All the images are resized to 512x512 pixels for streamlined
  analysis; this is accounted for in the physical spacing measurements.
* **Convert to HU** -- The entire image set is converted from its raw values to HU by applying the rescale intercept
  and slope which is contained in the DICOM properties.

**Analysis**

* **Determine phantom roll** -- Precise knowledge of the ROIs to analyze is important, and small changes in rotation
  could invalidate automatic results. The roll of the phantom is determined by examining the HU module and converting to
  binary. The air holes are then located and the angle of the two holes determines the phantom roll.

    .. note::
        For each step below, the "module" analyzed is actually the mean, median, or maximum of 3 slices (+/-1 slice around and
        including the nominal slice) to ensure robust measurements. Also, for each step/phantom module, the phantom center is
        determined, which corrects for the phantom pitch and yaw.

        Additionally, values tend to be lazy (computed only when asked for), thus the calculations listed may sometimes
        be performed only when asked for.

* **Determine HU linearity** -- The HU module (CTP404) contains several materials with different HU values. Using
  hardcoded angles (corrected for roll) and radius from the center of the phantom, circular ROIs are sampled which
  correspond to the HU material regions. The mean pixel values of these ROIs are the HU values.
* **Determine HU uniformity** -- HU uniformity (CTP486) is calculated in a very similar manner to HU linearity, but
  within the CTP486 module/slice.
* **Calculate Geometry/Scaling** -- The HU module (CTP404), besides HU materials, also contains several "nodes" which
  have an accurate spacing (5cm apart). Again, using hardcoded but corrected angles, the area around the 4 nodes are
  sampled and then a threshold is applied which identifies the node within the ROI sample. The center of this node is
  determined and then the space between nodes is calculated.
* **Calculate Spatial Resolution/MTF** -- The Spatial Resolution module (CTP528) contains 21 pairs of aluminum bars
  having varying thickness, which also corresponds to the thickness between the bars. One unique advantage of these
  bars is that they are all focused on and equally distant to the phantom center. Thus, several circular profiles are
  taken that go along all the line pairs. Starting on the proximal side of the line pairs, five profiles, each 1 pixel
  apart, sample an entire circle that cuts through the line pairs. The five profiles are concatenated and the median
  value extracted to effectively form one median profile. The peaks and valleys of the profile are located; peaks and
  valleys of the same line pair are averages. The relative MTF (i.e. normalized to the first line pair) is then
  calculated from these values.
* **Calculate Low Contrast Resolution** -- Not yet implemented, but it's being examined.

**Post-Analysis**

* **Test if values are within tolerance** -- For each module, the determined values are compared with the nominal values.
  If the difference between the two is below the specified tolerance then the module passes.


API Documentation
-----------------

The CBCT class uses several other classes. There are several Slices of Interest (SoI), most of which contain Regions of Interest (RoI).
SoIs have a base class as well as specialized classes for each specific slice.

.. autoclass:: pylinac.cbct.CBCT
    :members:
    :inherited-members:

Supporting Data Structure

.. autoclass:: pylinac.cbct.Algo_Data
    :members:

Slice Objects

.. autoclass:: pylinac.cbct.HU_Slice
    :members:

.. autoclass:: pylinac.cbct.Base_HU_Slice
    :members:

.. autoclass:: pylinac.cbct.UNIF_Slice
    :members:

.. autoclass:: pylinac.cbct.GEO_Slice
    :members:

.. autoclass:: pylinac.cbct.SR_Slice
    :members:

.. autoclass:: pylinac.cbct.Locon_Slice
    :members:

.. autoclass:: pylinac.cbct.Slice
    :members:

ROI Objects

.. autoclass:: pylinac.cbct.HU_ROI
    :members:

.. autoclass:: pylinac.cbct.GEO_ROI
    :members:

.. autoclass:: pylinac.cbct.SR_Circle_ROI
    :members:

.. autoclass:: pylinac.cbct.ROI_Disk
    :members:

.. autoclass:: pylinac.cbct.ROI
    :members:

.. autoclass:: pylinac.cbct.GEO_Line
    :members:
