## About
 
This repository demonstrates how Agfa PACS will obfuscate any DICOM images it touches. Agfa clearly believes in truth in advertising - their slogan for IMPAX is [`Because all PACS are not created equal`](http://www.agfahealthcare.com/he/germany/de/binaries/IMPAX_6_tcm602-90706.pdf). Indeed, while other PACS (Picture Archiving and Communication System) simply archive your images, Agfa will modify them in such a way that meta-data will be difficult to extract and impossible to modify with most popular tools.
 
To be clear, the method of obfuscation used by Agfa is legal DICOM. Specifically, the standard describes renaming of [Private Data Element Tags](http://dicom.nema.org/medical/dicom/current/output/chtml/part05/sect_7.8.html). However, this is a virtually unused feature. Further, most tools avoid renaming exisiting tags and use this mechanism only when wanting to ensure a new tag does not clash with an existing one. Therefore, the private tags used by manufacturers are typically reserved (in my experience, Agfa/dcm4che is the only exception). Therefore, support for this feature is limited. More importantly, the standard describes the usage of renaming for the `elimination of conflicts in Private Data Element Tags`. In sharp contrast, Agfa will take an input dataset that has unique tags, and rename them such that the resulting data will have a conflict.
 
The Agfa PACS uses the open source [dcm4che](https://www.dcm4che.org) PACS system. This is consistent with dcm4che's LGPL license and the Agfa team have contributed to the development of dcm4che. The renaming issue described here is common to both Agfa's commercial products and free implementations of dcm4che. See [here](https://github.com/rordenlab/dcm2niix/issues/437) for an example from the free implementation of dcm4che. Since the issues are common to both, I use the term Agfa to refer to any dcm4che-based implementation. However, I do think users of professional tools should have different expectations for support.
 
Note that any image touched by an Agfa PACS will be obfuscated, even if it is subsequently stored by a different PACS tool. Consider the example here: the original images were simple structures created by a Siemens scanner. The images were touched by an Agfa PACS but subsequently stored by [dcmtk](https://dicom.offis.de/dcmtk.php.en). Therefore, the Implementation Version Name (0002,0013) reports the name `OFFIS_DCMTK_361`. However, dcmtk's role is to simply copy the data, and it does not repair Agfa obfuscation. Since the implementation version name has been updated, the source of the disruption is not obvious (no tags in this data report `Agfa` or `dcm4che`). This interpretation and analysis are my own (Chris Rorden), based on the dataset provided by Joost P.A. Kuijer.
 
## An Example Dataset
 
The folder Siemens includes DICOM images exported directly from the MRI console to dcmtk. Looking at the meta-data (e.g. using [dcmdump](https://support.dcmtk.org/docs/dcmdump.html) or [gdcmdump](http://gdcm.sourceforge.net/html/gdcmdump.html)) reveals separate sequences are used for the series (0021,10xx) and image (0021,11xx) meta-data. There are no conflicting tags.
 
```
    (0021,0010) LO [SIEMENS MR SDS 01]                   #  18, 1 PrivateCreator
    (0021,10fe) SQ (Sequence with explicit length #=1)   # 106368, 1 Unknown Tag & Data
      (fffe,e000) na (Item with explicit length #=58)    # 106360, 1 Item
        (0021,0010) LO [SIEMENS MR SDS 01]               #  18, 1 PrivateCreator
        (0021,1001) IS [80]                              #   2, 1 Unknown Tag & Data
        (0021,1004) DS [1]                               #   2, 1 Unknown Tag & Data
        (0021,1045) CS [YES]                             #   4, 1 Unknown Tag & Data
...
    (0021,0011) LO [SIEMENS MR SDI 02]                   #  18, 1 PrivateCreator
    (0021,11fe) SQ (Sequence with explicit length #=1)   # 534, 1 Unknown Tag & Data
      (fffe,e000) na (Item with explicit length #=27)    # 526, 1 Item
        (0021,0011) LO [SIEMENS MR SDI 02]               #  18, 1 PrivateCreator
        (0021,1103) DS [249000]                          #   6, 1 Unknown Tag & Data
        (0021,1104) DS [0]                               #   2, 1 Unknown Tag & Data
        (0021,1145) SL 0\0\-1142                         #  12, 3 Unknown Tag & Data
...
```
 
The Agfa folder includes DICOM images exported from the MRI console to an Agfa PACS (running Agfa IMPAX Agility version 8.1 which leverages the [LGPL](https://dcm4che.atlassian.net/wiki/spaces/proj/pages/950309/License) dcm4che-3.3.7) and subsequently sent to dcmtk. Looking at the meta-data (e.g. using dcmdump) reveals that the image sequence (0021,11xx) is renamed (0021,10xx). This now conflicts with the tags of the series. Note that the private values (0021,11xx) were not renamed to avoid conflict with an Agfa tag: no tags are in the range (0021,11xx) in the entire dataset, while there is now a conflict of tags 0021,10xx. A tool must examine the text field of the private creator tag to detect the renaming. The opportunity for unintended consequences seems profound.
 
```
    (0021,0010) LO [SIEMENS MR SDS 01]                   #  18, 1 PrivateCreator
    (0021,10fe) SQ (Sequence with explicit length #=1)   # 106368, 1 Unknown Tag & Data
      (fffe,e000) na (Item with explicit length #=58)    # 106360, 1 Item
        (0021,0010) LO [SIEMENS MR SDS 01]               #  18, 1 PrivateCreator
        (0021,1001) IS [80]                              #   2, 1 Unknown Tag & Data
        (0021,1004) DS [1]                               #   2, 1 Unknown Tag & Data
        (0021,1004) DS [1]                               #   2, 1 Unknown Tag & Data
        (0021,1045) CS [YES]                             #   4, 1 Unknown Tag & Data
...
    (0021,0010) LO [SIEMENS MR SDI 02]                   #  18, 1 PrivateCreator
    (0021,10fe) SQ (Sequence with explicit length #=1)   # 534, 1 Unknown Tag & Data
      (fffe,e000) na (Item with explicit length #=27)    # 526, 1 Item
        (0021,0010) LO [SIEMENS MR SDI 02]               #  18, 1 PrivateCreator
        (0021,1003) DS [249000]                          #   6, 1 Unknown Tag & Data
        (0021,1004) DS [0]                               #   2, 1 Unknown Tag & Data
        (0021,1004) DS [0]                               #   2, 1 Unknown Tag & Data
        (0021,1045) SL 0\0\-1142                         #  12, 3 Unknown Tag & Data
...
```
 
## Consequences
 
To understand how this behavior will disrupt typical tools used by scientists, consider the important task of determining the `Number Of Prescans` (0021,1004) acquired. Regardless of the programming language used, the typical method to use will mimic the command line call to [dcmdump](https://support.dcmtk.org/docs/dcmdump.html) or [gdcmdump](http://gdcm.sourceforge.net/html/gdcmdump.html):
 
```
>dcmdump +F +P 0021,1004 ./Siemens/011-0001.img
# dcmdump (1/1): ./Siemens/011-0001.img
(0021,1004) DS [1]                                      #   2, 1 Unknown Tag & Data
```
However, the same call applied to the images touched by Agfa will not disambiguate between `Number Of Prescans` and `Time After Start` (0021,1104). One can also consider the reverse problem where the intention is to extract slice timing information from (0021,1104). Both values are importnat for fMRI studies used for research and surgical planning.
 
```
dcmdump +F +P 0021,1004 ./Agfa/011-0001.img
# dcmdump (1/1): ./Agfa/011-0001.img
(0021,1004) DS [1]                                      #   2, 1 Unknown Tag & Data
(0021,1004) DS [0]                                      #   2, 1 Unknown Tag & Data
(0021,1004) DS [0.05]                                   #   4, 1 Unknown Tag & Data
(0021,1004) DS [0.0975]                                 #   6, 1 Unknown Tag & Data
(0021,1004) DS [0.1475]                                 #   6, 1 Unknown Tag & Data
...
```
Note that after AGFA touches images, conflicting tags can have different [VRs](http://dicom.nema.org/dicom/2013/output/chtml/part05/sect_6.2.html), for example `BandwidthPerPixelPhaseEncode` (0021, 1153) is binary float (FD) while `Laterality4MF` (0021, 1053) is string (CS). Crucially, any subsequent tool can legally convert these images to implicit VR, which would compound the complexity of this renaming.
 
Users need to be very careful using private group numbers to remove or modify tags. Consider a user who wishes to add a serial number to the SiemensCoilForGradient2 (0021,1033) and [anonymize](https://www.fieldtriptoolbox.org/faq/how_can_i_anonymize_dicom_files/) the image by removing the private Siemens UsedPatientWeight (0021,1001) tag. These simple tools (like [gdcmanon](http://gdcm.sourceforge.net/html/gdcmanon.html) and [dcmodify](https://support.dcmtk.org/docs/dcmodify.html)) will not work as expected if the private groups have been renamed. The popular command line operations using private vendor tags will not work as expected. For example:
 
```
dcmodify -ea "(0021,1001)" -ea "(0021,1004)" 011-0001.dcm.dcm
```  
 
## Conclusion
 
I believe that Agfa's implementation does [conform to letter of the DICOM standard](http://dicom.nema.org/medical/dicom/current/output/chtml/part05/sect_7.8.html). However, it explicitly challenges the rationale and spirit of the standard, which is to eliminate tag conflicts rather than to create them. This is a limitation of the Agfa PACS, not of the MRI vendor's usage of private tags nor of dcm2niix's operation.
 
In an ideal world, all other tools would be upgraded to support Agfa's obfuscated but legal images. The [pydicom](https://github.com/pydicom/pydicom/blob/master/pydicom/_private_dict.py) code shows how this is implemented, for example `Diffusion B-Factor` has creator `Philips Imaging DD 001` and a variable tag (2001,xx03) rather than a fixed tag (2001,1003). Using this scheme provides robust DICOM support, albeit with consequences with regards to the simplicity and speed of the code. In practice, few tools appear to use this mechanism. Further, this would fix the symptom, and not the underlying problem. Popular methods for extracting and editing information from DICOM images will remain disrupted. Simply put, DICOM is a very complex standard, and these images store vital information. There are features of DICOM that are technically legal according to the standard that can have unintended consequences. Tools should strive to preserve simplicity where ever possible. Agfa is selling a commercial product, and I believe they have a responsibility to retain their customer's data. Until this issue is resolved, I would strongly encourage medical imaging centers to ensure that Agfa PACS do not touch their images.
 
I do not think these obfuscated images are of archival quality, and feel they will result in unintended consequences. I urge users to avoid using or purchasing these systems until this has been resolved. Users impacted by this should contact [Agfa](https://global.agfahealthcare.com/main/miscellaneous/interoperability/dicom_connectivity/), you have purchased a professional tool designed to handle vital health information and are entitled to a professional level of support.
