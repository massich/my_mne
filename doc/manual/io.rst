
.. contents:: Contents
   :local:
   :depth: 2

.. _ch_convert:

Importing data into MNE
~~~~~~~~~~~~~~~~~~~~~~~

This guide covers how to import data into MNE python. It includes instructions
for importing from common recording equipment in MEG and EEG, as well as how
to import raw data from numpy arrays.

Importing MEG data
##################

This section describes the data reading and conversion utilities included
with the MNE software. The cheatsheet below summarizes the different
file formats supported by MNE software.

===================   ========================   =========  =================================================================
Datatype              File format                Extension  MNE-Python function
===================   ========================   =========  =================================================================
MEG                   Elekta Neuromag            .fif       :func:`mne.io.read_raw_fif`
MEG                   4-D Neuroimaging / BTI      dir       :func:`mne.io.read_raw_bti`
MEG                   CTF                         dir       :func:`mne.io.read_raw_ctf`
MEG                   KIT                         sqd       :func:`mne.io.read_raw_kit` and :func:`mne.read_epochs_kit`
EEG                   Brainvision                .vhdr      :func:`mne.io.read_raw_brainvision`
EEG                   Neuroscan CNT              .cnt       :func:`mne.io.read_raw_cnt`
EEG                   European data format       .edf       :func:`mne.io.read_raw_edf`
EEG                   Biosemi data format        .bdf       :func:`mne.io.read_raw_edf`
EEG                   General data format        .gdf       :func:`mne.io.read_raw_edf`
EEG                   EGI simple binary          .egi       :func:`mne.io.read_raw_egi`
EEG                   EGI MFF format             .mff       :func:`mne.io.read_raw_egi`
EEG                   EEGLAB                     .set       :func:`mne.io.read_raw_eeglab` and :func:`mne.read_epochs_eeglab`
EEG                   eXimia                     .nxe       :func:`mne.io.read_raw_eximia`
Electrode locations   elc, txt, csd, sfp, htps   Misc       :func:`mne.channels.read_montage`
Electrode locations   EEGLAB loc, locs, eloc     Misc       :func:`mne.channels.read_montage`
MEG and EEG           FieldTrip raw/epochs/avg   .mat       :func:`mne.io.read_raw_fieldtrip`, :func:`mne.read_epochs_fieldtrip` and :func:`mne.read_evoked_fieldtrip`
===================   ========================   =========  =================================================================

.. note::
    All IO functions in MNE-Python performing reading/conversion of MEG and
    EEG data can be found in :mod:`mne.io` and start with `read_raw_`. All
    supported data formats can be read in MNE-Python directly without first
    saving it to fif.

.. note::
    Irrespective of the units used in your manufacturer's format, MNE-Python
    will always use the units listed below and perform conversions during the
    IO procedure if necessary.

    * V: eeg, eog, seeg, emg, ecg, bio, ecog
    * T: mag
    * T/m: grad
    * M: hbo, hbr
    * Am: dipole
    * AU: misc

.. note::
    MNE-Python performs all computation in memory using the double-precision
    64-bit floating point format. This means that the data is typecasted into
    `float64` format as soon as it is read into memory. The reason for this is
    that operations such as filtering, preprocessing etc. are more accurate when
    using the double-precision format. However, for backward compatibility, it
    writes the `fif` files in a 32-bit format by default. This is advantageous
    when saving data to disk as it consumes less space.

    However, if the users save intermediate results to disk, they should be aware
    that this may lead to loss in precision. The reason is that writing to disk is
    32-bit by default and then typecasting to 64-bit does not recover the lost
    precision. In case you would like to retain the 64-bit accuracy, there are two
    possibilities:

    * Chain the operations in memory and not save intermediate results
    * Save intermediate results but change the ``dtype`` used for saving. However,
      this may render the files unreadable in other software packages

Elekta NeuroMag (.fif)
======================

Neuromag Raw FIF files can be loaded using :func:`mne.io.read_raw_fif`.

.. note::
    If the data were recorded with MaxShield on and have not been processed
    with MaxFilter, they may need to be loaded with
    ``mne.io.read_raw_fif(..., allow_maxshield=True)``.

Importing 4-D Neuroimaging / BTI data
=====================================

MNE-Python includes the :func:`mne.io.read_raw_bti` to read and convert 4D / BTI data.
This reader function will by default replace the original channel names,
typically composed of the letter `A` and the channel number with Neuromag.
To import the data, the following input files are mandatory:

- A data file (typically c,rfDC)
  containing the recorded MEG time-series.

- A hs_file
  containing the digitizer data.

- A config file
  containing acquisition information and metadata.

By default :func:`mne.io.read_raw_bti` assumes these three files to be located
in the same folder.

.. note:: While reading the reference or compensation channels,
          currently, the compensation weights are not processed.
          As a result, the :class:`mne.io.Raw` object and the corresponding fif
          file does not include information about the compensation channels
          and the weights to be applied to realize software gradient
          compensation. To augment the Magnes fif files with the necessary
          information, the command line tools include the utilities
          :ref:`mne_create_comp_data`, and :ref:`mne_add_to_meas_info`.
          Including the compensation channel data is recommended but not
          mandatory. If the data are saved in the Magnes system are already
          compensated, there will be a small error in the forward calculations
          whose significance has not been evaluated carefully at this time.


Creating software gradient compensation data
--------------------------------------------

The utility mne_create_comp_data was
written to create software gradient compensation weight data for
4D Magnes fif files. This utility takes a text file containing the
compensation data as input and writes the corresponding fif file
as output. This file can be merged into the fif file containing
4D Magnes data with the utility :ref:`mne_add_to_meas_info`.
See :ref:`mne_create_comp_data` for command-line options.


Importing CTF data
==================

In MNE-Python, :func:`mne.io.read_raw_ctf` can be used to read CTF data.


Importing CTF Polhemus data
===========================

The CTF MEG systems store the Polhemus digitization data
in text files. The utility :ref:`mne_ctf_dig2fiff` was
created to convert these data files into the fif and hpts formats.


.. _BEHDDFBI:

Applying software gradient compensation
---------------------------------------

Since the software gradient compensation employed in CTF
systems is a reversible operation, it is possible to change the
compensation status of CTF data in the data files as desired. This
section contains information about the technical details of the
compensation procedure and a description of mne_compensate_data ,
which is a utility to change the software gradient compensation
state in evoked-response data files.

The fif files containing CTF data converted using the utility mne_ctf2fiff contain
several compensation matrices which are employed to suppress external disturbances
with help of the reference channel data. The reference sensors are
located further away from the brain than the helmet sensors and
are thus measuring mainly the external disturbances rather than magnetic
fields originating in the brain. Most often, a compensation matrix
corresponding to a scheme nicknamed *Third-order gradient
compensation* is employed.

Let us assume that the data contain :math:`n_1` MEG
sensor channels, :math:`n_2` reference sensor
channels, and :math:`n_3` other channels.
The data from all channels can be concatenated into a single vector

.. math::    x = [x_1^T x_2^T x_3^T]^T\ ,

where :math:`x_1`, :math:`x_2`,
and :math:`x_3` are the data vectors corresponding
to the MEG sensor channels, reference sensor channels, and other
channels, respectively. The data before and after compensation,
denoted here by :math:`x_{(0)}` and :math:`x_{(k)}`, respectively,
are related by

.. math::    x_{(k)} = M_{(k)} x_{(0)}\ ,

where the composite compensation matrix is

.. math::    M_{(k)} = \begin{bmatrix}
		I_{n_1} & C_{(k)} & 0 \\
		0 & I_{n_2} & 0 \\
		0 & 0 & I_{n_3}
		\end{bmatrix}\ .

In the above, :math:`C_{(k)}` is a :math:`n_1` by :math:`n_2` compensation
data matrix corresponding to compensation "grade" :math:`k`.
It is easy to see that

.. math::    M_{(k)}^{-1} = \begin{bmatrix}
		I_{n_1} & -C_{(k)} & 0 \\
		0 & I_{n_2} & 0 \\
		0 & 0 & I_{n_3}
		\end{bmatrix}\ .

To convert from compensation grade :math:`k` to :math:`p` one
can simply multiply the inverse of one compensate compensation matrix
by another and apply the product to the data:

.. math::    x_{(k)} = M_{(k)} M_{(p)}^{-1} x_{(p)}\ .

This operation is performed by :meth:`mne.io.Raw.apply_gradient_compensation`.


Importing KIT MEG system data
=============================

MNE-Python includes the :func:`mne.io.read_raw_kit` and
:func:`mne.read_epochs_kit` to read and convert KIT MEG data.
This reader function will by default replace the original channel names,
which typically with index starting with zero, with ones with an index starting with one.

To import continuous data, only the input .sqd or .con file is needed. For epochs,
an Nx3 matrix containing the event number/corresponding trigger value in the
third column is needed.

The following input files are optional:

- A KIT marker file (mrk file) or an array-like
  containing the locations of the HPI coils in the MEG device coordinate system.
  These data are used together with the elp file to establish the coordinate
  transformation between the head and device coordinate systems.

- A Polhemus points file (elp file) or an array-like
  containing the locations of the fiducials and the head-position
  indicator (HPI) coils. These data are usually given in the Polhemus
  head coordinate system.

- A Polhemus head shape data file (hsp file) or an array-like
  containing locations of additional points from the head surface.
  These points must be given in the same coordinate system as that
  used for the elp file.


.. note:: The output fif file will use the Neuromag head coordinate system convention, see :ref:`BJEBIBAI`. A coordinate transformation between the Polhemus head coordinates and the Neuromag head coordinates is included.


By default, KIT-157 systems assume the first 157 channels are the MEG channels,
the next 3 channels are the reference compensation channels, and channels 160
onwards are designated as miscellaneous input channels (MISC 001, MISC 002, etc.).
By default, KIT-208 systems assume the first 208 channels are the MEG channels,
the next 16 channels are the reference compensation channels, and channels 224
onwards are designated as miscellaneous input channels (MISC 001, MISC 002, etc.).

In addition, it is possible to synthesize the digital trigger channel (STI 014)
from available analog trigger channel data by specifying the following parameters:

- A list of trigger channels (stim) or default triggers with order: '<' | '>'
  Channel-value correspondence when converting KIT trigger channels to a
  Neuromag-style stim channel. By default, we assume the first eight miscellaneous
  channels are trigger channels. For '<', the largest values are assigned
  to the first channel (little endian; default). For '>', the largest values are
  assigned to the last channel (big endian). Can also be specified as a list of
  trigger channel indexes.
- The trigger channel slope (slope) : '+' | '-'
  How to interpret values on KIT trigger channels when synthesizing a
  Neuromag-style stim channel. With '+', a positive slope (low-to-high)
  is interpreted as an event. With '-', a negative slope (high-to-low)
  is interpreted as an event.
- A stimulus threshold (stimthresh) : float
  The threshold level for accepting voltage changes in KIT trigger
  channels as a trigger event.

The synthesized trigger channel data value at sample :math:`k` will
be:

.. math::    s(k) = \sum_{p = 1}^n {t_p(k) 2^{p - 1}}\ ,

where :math:`t_p(k)` are the thresholded
from the input channel data d_p(k):

.. math::    t_p(k) = \Bigg\{ \begin{array}{l}
		 0 \text{  if  } d_p(k) \leq t\\
		 1 \text{  if  } d_p(k) > t
	     \end{array}\ .

The threshold value :math:`t` can
be adjusted with the ``stimthresh`` parameter, see below.


Importing EEG data
##################

The MNE package includes various functions and utilities for reading EEG
data and electrode templates.

BrainVision (.vhdr, .vmrk, .eeg)
================================

The BrainVision file format consists out of three separate files:

1. A text header file (``.vhdr``) containing meta data
2. A text marker file (``.vmrk``) containing information about events in the
   data
3. A binary data file (``.eeg``) containing the voltage values of the EEG

Both text files are based on the
`Microsoft Windows INI format <https://en.wikipedia.org/wiki/INI_file>`_
consisting of:

* sections marked as ``[square brackets]``
* comments marked as ``; comment``
* key-value pairs marked as ``key=value``

A documentation for core BrainVision file format is provided by Brain Products.
You can view the specification
`here <https://docs.google.com/viewer?url=https://raw.githubusercontent.com/sappelhoff/brainvision-validator/master/doc/BrainVisionCoreFileFormat.pdf>`_

BrainVision EEG files can be read in using :func:`mne.io.read_raw_brainvision`
with the .vhdr header file as an input.

.. warning:: Renaming BrainVision files can be problematic due to their
             multifile structure. See this
             `example <https://mne-tools.github.io/mne-bids/auto_examples/rename_brainvision_files>`_
             for an instruction.


European data format (.edf)
===========================

EDF and EDF+ files can be read in using :func:`mne.io.read_raw_edf`.

`EDF (European Data Format) <http://www.edfplus.info/specs/edf.html>`_ and
`EDF+ <http://www.edfplus.info/specs/edfplus.html>`_ are 16-bit formats.

The EDF+ files may contain an annotation channel which can be used to store
trigger information. The Time-stamped Annotation Lists (TALs) on the
annotation  data can be converted to a trigger channel (STI 014) using an
annotation map file which associates an annotation label with a number on
the trigger channel.

Saving EDF files is not supported natively (mne 0.18) yet.
This `gist <https://gist.github.com/skjerns/bc660ef59dca0dbd53f00ed38c42f6be>`__
can be used to save any mne.io.Raw into EDF/EDF+/BDF/BDF+.

Biosemi data format (.bdf)
==========================

The `BDF format <http://www.biosemi.com/faq/file_format.htm>`_ is a 24-bit
variant of the EDF format used by the EEG systems manufactured by a company
called BioSemi. It can also be read in using :func:`mne.io.read_raw_edf`.

.. warning:: The data samples in a BDF file are represented in a 3-byte (24-bit) format. Since 3-byte raw data buffers are not presently supported in the fif format these data will be changed to 4-byte integers in the conversion.

General data format (.gdf)
==========================

GDF files can be read in using :func:`mne.io.read_raw_edf`.

`GDF (General Data Format) <https://arxiv.org/abs/cs/0608052>`_ is a flexible
format for biomedical signals, that overcomes some of the limitations of the
EDF format. The original specification (GDF v1) includes a binary header,
and uses an event table. An updated specification (GDF v2) was released in
2011 and adds fields for additional subject-specific information (gender,
age, etc.) and allows storing several physical units and other properties.
Both specifications are supported in MNE.

Neuroscan CNT data format (.cnt)
================================

CNT files can be read in using :func:`mne.io.read_raw_cnt`.
The channel locations can be read from a montage or the file header. If read
from the header, the data channels (channels that are not assigned to EOG, ECG,
EMG or misc) are fit to a sphere and assigned a z-value accordingly. If a
non-data channel does not fit to the sphere, it is assigned a z-value of 0.
See :ref:`BJEBIBAI`

.. warning::
    Reading channel locations from the file header may be dangerous, as the
    x_coord and y_coord in ELECTLOC section of the header do not necessarily
    translate to absolute locations. Furthermore, EEG-electrode locations that
    do not fit to a sphere will distort the layout when computing the z-values.
    If you are not sure about the channel locations in the header, use of a
    montage is encouraged.

EGI simple binary (.egi)
========================

EGI simple binary files can be read in using :func:`mne.io.read_raw_egi`.
The EGI raw files are simple binary files with a header and can be exported
from using the EGI Netstation acquisition software.


EEGLAB set files (.set)
=======================

EEGLAB .set files can be read in using :func:`mne.io.read_raw_eeglab`
and :func:`mne.read_epochs_eeglab`.

Importing EEG data saved in the Tufts University format
=======================================================

The command line utility :ref:`mne_tufts2fiff` was
created in collaboration with Phillip Holcomb and Annette Schmid
from Tufts University to import their EEG data to the MNE software.

The Tufts EEG data is included in three files:

- The raw data file containing the acquired
  EEG data. The name of this file ends with the suffix ``.raw`` .

- The calibration raw data file. This file contains known calibration
  signals and is required to bring the data to physical units. The
  name of this file ends with the suffix ``c.raw`` .

- The electrode location information file. The name of this
  file ends with the suffix ``.elp`` .

See the options for the command-line utility :ref:`mne_tufts2fiff`.

eXimia EEG data
===============

EEG data from the Nexstim eXimia system can be read in using the
:func:`mne.io.read_raw_eximia` function.

Setting EEG references
######################

The preferred method for applying an EEG reference in MNE is
:func:`mne.set_eeg_reference`, or equivalent instance methods like
:meth:`raw.set_eeg_reference() <mne.io.Raw.set_eeg_reference>`. By default,
an average reference is used. Instead of applying the average reference to
the data directly, an average EEG reference projector is created that is
applied like any other SSP projection operator.

There are also other functions that can be useful for other referencing
operations. See :func:`mne.set_bipolar_reference` and
:func:`mne.add_reference_channels` for more information.


Reading Electrode locations and Headshapes for EEG recordings
#############################################################

Some EEG formats (EGI, EDF/EDF+, BDF) neither contain electrode location
information nor head shape digitization information. Therefore, this information
has to be provided separately. For that purpose all readers have a montage
parameter to read locations from standard electrode templates or a polhemus
digitizer file. This can also be done post-hoc using the
:func:`mne.io.Raw.set_montage` method of the Raw object in memory.


When using the locations of the fiducial points the digitization data
are converted to the MEG head coordinate system employed in the
MNE software, see :ref:`BJEBIBAI`.


Creating MNE data structures from arbitrary data (from memory)
##############################################################

Arbitrary (e.g., simulated or manually read in) raw data can be constructed
from memory by making use of :class:`mne.io.RawArray`, :class:`mne.EpochsArray`
or :class:`mne.EvokedArray` in combination with :func:`mne.create_info`.

This functionality is illustrated in :ref:`sphx_glr_auto_examples_io_plot_objects_from_arrays.py`.
Using 3rd party libraries such as NEO (https://github.com/NeuralEnsemble/python-neo) in combination
with these functions abundant electrophysiological file formats can be easily loaded
into MNE.


Importing MEG/EEG data from FieldTrip
#####################################

MNE-Python includes :func:`mne.io.read_raw_fieldtrip`, :func:`mne.read_epochs_fieldtrip` and :func:`mne.read_evoked_fieldtrip` to read data coming from FieldTrip.

The data is imported directly from a ``.mat`` file.

The ``info`` parameter can be explicitly set to ``None``. The import functions will still work but:

#. All channel locations will be in head coordinates.
#. Channel orientations cannot be guaranteed to be accurate.
#. All channel types will be set to generic types.

This is probably fine for anything that does not need that information, but if you intent to do things like interpolation of missing channels, source analysis or look at the RMS pairs of planar gradiometers, you most likely run into problems.

It is **highly recommended** to provide the ``info`` parameter as well. The ``info`` dictionary can be extracted by loading the original raw data file with the corresponding MNE-Python functions::

    original_data = mne.io.read_raw_fiff('original_data.fif', preload=False)
    original_info = original_data.info
    data_from_ft = mne.read_evoked_fieldtrip('evoked_data.mat', original_info)

The imported data can have less channels than the original data. Only the information for the present ones is extracted from the ``info`` dictionary.

As of version 0.17, importing FieldTrip data has been tested on a variety of systems with the following results:

+----------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+
| System   | Read Raw Data                                                                | Read Epoched Data                                                            | Read Evoked Data                                                             |
+==========+==============================================================================+==============================================================================+==============================================================================+
| BTI      | Works                                                                        | Untested                                                                     | Untested                                                                     |
+----------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+
| CNT      | Data imported as microvolts. Otherwise fine.                                 | Data imported as microvolts. Otherwise fine.                                 | Data imported as microvolts. Otherwise fine.                                 |
+----------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+
| CTF      | Works                                                                        | Works                                                                        | Works                                                                        |
+----------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+
| EGI      | Mostly Ok.Data imported as microvolts. FieldTrip does not apply calibration. | Mostly Ok.Data imported as microvolts. FieldTrip does not apply calibration. | Mostly Ok.Data imported as microvolts. FieldTrip does not apply calibration. |
+----------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+
| KIT      | Does not work. Channel names are different in MNE-Python and FieldTrip       | Does not work. Channel names are different in MNE-Python and FieldTrip       | Does not work. Channel names are different in MNE-Python and FieldTrip       |
+----------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+
| Neuromag | Works                                                                        | Works                                                                        | Works                                                                        |
+----------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+
| eximia   | Work                                                                         | Untested                                                                     | Untested                                                                     |
+----------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+------------------------------------------------------------------------------+
