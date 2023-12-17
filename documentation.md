Documentation of model structure project
Arnav Nagle
nagle@berkeley.edu

### Summary
The purpose of this project is to use structural and sequence comparison to identify SCOPe domains that are related to a user-inputted human AlphaFold model and present information about each comparison to the user on the SCOPe website.  So far, I have implemented a prototype (note: see below for known issues) for one model structure. The prototype incorporates SQL tables, Java code to populate these tables, and a PHP landing page (with inline JavaScript and modifications to the TypeScript RCSB Mol\* library). 


### Recreating all model structure tables from scratch

1. Download the human model structures from the AlphaFold website. (Note: AlphaFoldDB has irregular releases, and the latest at the time of writing is v4. See https://alphafold.ebi.ac.uk/faq for release notes.)

mkdir af2\_human\_v4
cd af2\_human\_v4
curl https://ftp.ebi.ac.uk/pub/databases/alphafold/latest/UP000005640\_9606\_HUMAN\_v4.tar | tar x

2. Convert the model structures to xml. (Note: I have not written code that parses the xml files yet; I am currently parsing the cif files, which have equivalent information, with the RCSB's cif-tools. We ultimately want to switch to using xml only to avoid this dependency.)

```
# TODO: I've written a Java script that does this at /h/anagle/populate\_model\_structure/MakeAf2Dir.java . I was in the middle of reworking this to copy rather than unzip the gzipped files. Commit b81e4 of the Git repo rooted at /h/anagle/populate\_model\_structure should work, but it makes all the files unzipped.
```

3. Create the model\_structure\_source, model\_structure, and model\_structure\_uniprot tables.

mysql -h doppelbock scop

CREATE TABLE `model\_structure\_source` (
  `id` int(10) unsigned NOT NULL AUTO\_INCREMENT,
  `description` text NOT NULL,
  `date` date NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 COMMENT='Publication source for theoretical protein structure model';

insert into model\_structure\_source (description, date) values ("AlphaFoldDB v4", 20220930);

CREATE TABLE `model\_structure` (
  `id` int(10) unsigned NOT NULL AUTO\_INCREMENT,
  `source\_id` int(10) unsigned NOT NULL,
  `pdb\_path` text DEFAULT NULL COMMENT 'absolute path to file',
  `xml\_path` text DEFAULT NULL COMMENT 'absolute path to xml',
  `cif\_path` text DEFAULT NULL COMMENT 'absolute path to cif',
  PRIMARY KEY (`id`),
  KEY `source\_id` (`source\_id`),
  CONSTRAINT `model\_structure\_ibfk\_1` FOREIGN KEY (`source\_id`) REFERENCES `model\_structure\_source` (`id`) ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=latin1 COMMENT='Published theoretical models of protein structures';


CREATE TABLE `model\_structure\_uniprot` (
  `model\_structure\_id` int(10) unsigned NOT NULL,
  `uniprot\_id` int(10) unsigned NOT NULL,
  `uniprot\_start` int(5) unsigned COMMENT '0-indexed, inclusive',
  `uniprot\_end` int(5) unsigned COMMENT '0-indexed, exclusive',
  KEY `model\_structure\_id` (`model\_structure\_id`),
  KEY `uniprot\_id` (`uniprot\_id`),
  CONSTRAINT `model\_structure\_uniprot\_ibfk\_1` FOREIGN KEY (`model\_structure\_id`) REFERENCES `model\_structure` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `model\_structure\_uniprot\_ibfk\_2` FOREIGN KEY (`uniprot\_id`) REFERENCES `uniprot` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=latin1 COMMENT='Uniprot mappings for sequences of theoretical protein structure models';

4. Populate model\_structure and model\_structure\_uniprot. Note: Modify the file paths in MakeDBModelStructure.java to the directories where you stored the cif, pdb, and xml files respectively. Check that the dates in the insert statement are correct. Ensure that the correct version of each Uniprot sequence is imported into the uniprot table before running.

cd /h/anagle/populate\_model\_structure/
/h/anagle/jdk/jdk-18.0.2/bin/java MakeDBModelStructure

/h/anagle/jdk/jdk-18.0.2/bin/java MakeDBModelStructureUniprot

5. Create the structure\_aligner and model\_vs\_domain\_structure\_alignment table.
CREATE TABLE `structure\_aligner` (
  `id` int(10) unsigned NOT NULL AUTO\_INCREMENT,
  `name` text NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 COMMENT='Structural aligner program description'

insert into structure\_aligner (name) values ("CE");

CREATE TABLE `model\_vs\_domain\_structure\_alignment` (
  `id` int(10) unsigned NOT NULL AUTO\_INCREMENT,
  `model\_id` int(10) unsigned NOT NULL,
  `domain\_id` int(10) unsigned NOT NULL,
  `structure\_aligner\_id` int(10) unsigned NOT NULL,
  `p\_value` double,
  `rmsd` double COMMENT 'root-mean-square deviation in angstroms',
  `z\_score` double DEFAULT NULL,	
  `num\_eq\_res` int(5) DEFAULT NULL COMMENT 'Number of equivalent residues',
  `model\_start` int(10) unsigned NOT NULL COMMENT '0-indexed into model sequence, inclusive',
  `model\_end` int(10) unsigned NOT NULL COMMENT '0-indexed into model sequence, inclusive',
  `domain\_start\_ATOM` int(10) unsigned NOT NULL COMMENT '0-indexed into domain ATOM records, inclusive',
  `domain\_end\_ATOM` int(10) unsigned NOT NULL COMMENT '0-indexed into whole chain RAF, inclusive',
  `domain\_start\_raf` int(10) unsigned NOT NULL COMMENT '0-indexed into whole chain RAF, inclusive',
  `domain\_end\_raf` int(10) unsigned NOT NULL COMMENT '0-indexed into whole chain RAF end of hit, inclusive',
  `domain\_start\_SEQRES` int(10) unsigned NOT NULL COMMENT '0-indexed into SEQRES, inclusive',
  `domain\_end\_SEQRES` int(10) unsigned NOT NULL COMMENT '0-indexed into SEQRES, inclusive',
  `translate\_x` double NOT NULL COMMENT 'x-coordinate translation from model to domain',
  `translate\_y` double NOT NULL COMMENT 'y-coordinate translation from model to domain',
  `translate\_z` double NOT NULL COMMENT 'z-coordinate translation from model to domain',
  `rotate\_1\_1` double NOT NULL COMMENT '1st row, 1st col (1-indexed) in rotation matrix from model to domain',
  `rotate\_1\_2` double NOT NULL COMMENT '1st row, 2nd col (1-indexed) in rotation matrix from model to domain',
  `rotate\_1\_3` double NOT NULL COMMENT '1st row, 3rd col (1-indexed) in rotation matrix from model to domain',
  `rotate\_2\_1` double NOT NULL COMMENT '2nd row, 1st col (1-indexed) in rotation matrix from model to domain',
  `rotate\_2\_2` double NOT NULL COMMENT '2nd row, 2nd col (1-indexed) in rotation matrix from model to domain',
  `rotate\_2\_3` double NOT NULL COMMENT '2nd row, 3rd col (1-indexed) in rotation matrix from model to domain',
  `rotate\_3\_1` double NOT NULL COMMENT '3rd row, 1st col (1-indexed) in rotation matrix from model to domain',
  `rotate\_3\_2` double NOT NULL COMMENT '3rd row, 2nd col (1-indexed) in rotation matrix from model to domain',
  `rotate\_3\_3` double NOT NULL COMMENT '3rd row, 3rd col (1-indexed) in rotation matrix from model to domain',
  PRIMARY KEY (`id`),
  KEY `model\_id` (`model\_id`),
  KEY `domain\_id` (`domain\_id`),
  KEY `structure\_aligner\_id` (`structure\_aligner\_id`),
  CONSTRAINT `model\_vs\_domain\_structure\_alignment\_ibfk\_1` FOREIGN KEY (`model\_id`) REFERENCES `model\_structure` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `model\_vs\_domain\_structure\_alignment\_ibfk\_2` FOREIGN KEY (`domain\_id`) REFERENCES `astral\_domain` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `model\_vs\_domain\_structure\_alignment\_ibfk\_3` FOREIGN KEY (`structure\_aligner\_id`) REFERENCES `structure\_aligner` (`id`) ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=latin1 COMMENT='Structural alignments between model structures and SCOP domains'

6. Populate model\_vs\_structure\_alignment.
Usage:
```
# TODO: Write a bash script that iterates over all model structure IDs and submits them as jobs to GridEngine with the qsub command. Current code for one model structure is at /h/anagle/populate\_model\_vs\_structure
```

### Landing page
The landing page code I wrote is at /h/anagle/www/scop-boot/alphafold/alphafold.php
This requires you to build the modified RCSB package I've put here: https://github.com/ArnNag/rcsb-molstar/tree/make\_component
To access the test model I've been testing on, go to https://strgen.org/~anagle/scop-boot/alphafold/alphafold.php?name=AF-P63000-F1
(This is referred to in my notebook as pdb\_rowcol\_fill.php)
```
# TODO: the test case landing page is not currently working. I think it's due to hardcoded references to the paths of the structure files that we need to change to the final location we made in /lab/db .
```

### Search functionality
We will eventually need to implement search functionality where users can input a Uniprot accession number. This will require a landing page for Uniprot accessions and all associated model structures. There are some Uniprot accessions numbers in the AlphaFoldDB download that have several "fragments" because the sequence was too long to generate at once (See https://alphafold.ebi.ac.uk/download).

### Mapping PDB to Uniprot
We eventually want to show the user all PDB entries associated with a Uniprot entry on the Uniprot entry's landing page. These associations are denoted in PDB files with the DBREF records, and in CIF files in the STRUCT\_REF category. We store the DBREF records mapped to RAF indices in the pdb\_chain\_dbref table. PDB DBREF records may contain errors and do not specify which Uniprot version they refer to, so I wrote code that uses the Levenshtein distance to see whether the region of the Uniprot sequence referred to by the DBREF records approximately matches the PDB chain sequence. 

```
# TODO: Usage, output format
```

If we want to do analyses about where SCOPe domains lie in AlphaFold predictions for the same sequence as the PDB chain the domain derives from (say, to examine the plDDT and PAE within the domain), we need to be careful about which index in the PDB sequence corresponds to which index in the Uniprot sequence. The alignments between PDB chains and Uniprot sequences are theoretically provided by the DBREF records, with differences documented in SEQADV (STRUCT\_REF\_SEQ and STRUCT\_REF\_SEQ\_DIF in CIF). SIFTS mappings are supposed to solve this problem, and are included as part of the "updated" CIF files that you can download from the PDBe website (not RCSB). The relevant records are in the ATOM\_SITE and PDBX\_SIFTS\_UNP\_SEGMENTS categories. I still have not found any record that explicitly references the version of the Uniprot sequence.

### Known issues
1. The original Mol\* viewer has a sequence view that allows you to view the full SEQRES sequence and click on the residues in the sequence to locate them in the structure and select regions of sequence in Selection Mode. This disappears when forcing the viewer into a div.
```
# TODO: link to what this is supposed to look like, describe how it disappeared
```
2. The AlphaFold model is colored by PAE by default, but when I make a component of the hit region, it becomes orange.
3. Unstable awk code
4. The BLAST gaps are not accounted for correctly when computing the boundaries of hits.


