## About

This repository demonstrates how Agfa PACS will obfuscate any DICOM images it touches. Agfa clearly believes in truth in advertising - their slogan for IMPAX is `[Because all PACS are not created equal](http://www.agfahealthcare.com/he/germany/de/binaries/IMPAX_6_tcm602-90706.pdf).` Indeed, while other PACS (Picture Archiving and Communication System) simply archives your images, Agfa will modify them in such a way that meta data will be difficult to extract and impossible to modify with most popular tools. 

To be clear, the method of obfuscation used by Agfa is legal in recent versions of the DICOM standard. Specifically, the standard describes renaming of [Private Data Element Tags](http://dicom.nema.org/medical/dicom/current/output/chtml/part05/sect_7.8.html). However, this is a new and virtually unused feature, meaning support for it is limited. More importantly, the standard describes the usage of renaming for the `elimination of conflicts in Private Data Element Tags`. In sharp contrast, Agfa will take an input dataset that has unique tags, and rename them such that the resulting data will have a conflict. Such behavior will disrupt simple tools that attempt to extract of rename tags. 

In theory, open source tools like dcm2niix could be upgraded to support Agfa's obfuscated images. However, this would fix the symptom, and not the underlying problem. All users would be impacted by slower conversion speeds, regardless of whether they use Agfa. Developers would be saddled with code that is more difficult to maintain. Worse, other tools and common simple methods for editing DICOMs will remain disrupted by Agfa's approach. Simply put, Agfa's Picture Archiving and Communication System corrupts images so they are no longer of archival quality. DICOM is a very complex standard, and these images store vital information. There are features of DICOM that are technically legal according to the standard that can have unintended consequences. Tools should strive to preserve simplicity where ever possible. Agfa is selling a commercial product, and I believe they have a responsibility to retain their customer's data. Until this issue is resolved, I would strongly encourage centers ensure Agfa PACS do not touch their images. 

Note that any image touched by an Agfa PACS will be obfuscated, even if it is subsequently stored by a different PACS tool. Consider the example here: the original images were simple structures created by a Siemens scanner. The images were touched by an Agfa PACS but subsequently stored by [dcmtk](https://dicom.offis.de/dcmtk.php.en). Therefore, the Implementation Version Name (0002,0013) reports the name `OFFIS_DCMTK_361`. However, dcmtk's role is to simply copy the data, and it does not repair Agfa obfuscation. Since the implementation version name has been updated, the source of the disruption is not obvious (no tags in this data report `Agfa` or `IMPAX`.

## An Example Dataset

The folder Siemens includes DICOM images exported directly from the MRI console to dcmtk. Looking at the meta data (e.g. using dcmdump) reveals separate blocks for the series (0021,10xx) and the images (0021,11xx). There are no conflicts of tags.

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

The Agfa folder includes DICOM images exported from the MRI console to an Agfa PACS and subsequently to dcmtk. Looking at the meta data (e.g. using dcmdump) reveals that the image sequence (0021,11xx) is renamed (0021,11xx). This now conflicts with the tags of the series. Note that the private values (0021,11xx) were not renamed to avoid conflict with an Agfa tag: no tags in the series (0021,11xx) exist in the entire dataset, while there is duplication of tags 0021,10xx, even where different VRs are used. A tool must examine the text field of the private creator tag to detect the renaming. Remember, any subsequent tool can legally convert these images to implicit VR. The opportunity for unintended consequences seems profound.

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

To understand how this behavior will disrupt typical tools used by scientist, consider the important task of determining the `Number Of Prescans` (0021,1004) acquired. This is a vital bit of information that many tools will want to automatically determine. Regardless of the programming language used, the typical method to use will mimic the command line call:

``` 
>dcmdump +F +P 0021,1004 ./Siemens/011-0001.img
# dcmdump (1/1): ./Siemens/011-0001.img
(0021,1004) DS [1]                                      #   2, 1 Unknown Tag & Data
```
However, the same call applied to the images touched by Agfa will not disambiguate between `Number Of Prescans` and `Time After Start` (0021,1104). One can also consider the reverse problem where the intention is to extract slice timing information from (0021,1104). Note that after AGFA touches images, conflicting tags can have different [VRs](http://dicom.nema.org/dicom/2013/output/chtml/part05/sect_6.2.html), for example `BandwidthPerPixelPhaseEncode` (0021, 1153) is binary float (FD) while `Laterality4MF` (0021, 1053) is string (CS).

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

## Conclusion

I believe that Agfa's implementation does [conform to letter of the DICOM standard](http://dicom.nema.org/medical/dicom/current/output/chtml/part05/sect_7.8.html). However, it explicitly challenges the rationale and spirit of the standard, which is to eliminate tag conflicts rather than to create them. This is a limitation of the Agfa PACS, not of the MRI vendor's usage of private tags nor of dcm2niix's operation. I do not think these obfuscated images are of archival quality, and feel certain they will result in widespread unintended consequences. I urge users to avoid using or purchasing these systems until this has been resolved. Users impacted by this should contact [Agfa](https://global.agfahealthcare.com/main/miscellaneous/interoperability/dicom_connectivity/), you have purchased a professional tool designed to handle vital health information and are entitled to a professional level of support.
