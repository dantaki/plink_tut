# Plink Tutorial for VCF files

## Notes:

Plink is old-school. It was written when microarrays were the bread and butter of genomic interrogation. In microarray analysis each marker has a unique identifier. This unique identifier is in plink instead of genomic position.

However, with WES/WGS, VCF files do not have unique indetifiers (in the "ID" column). Many variants not present in dbSNP will have a "." entry. Therefore, **we must create unique indentifiers for each variant** for the VCF to be used in plink.

Why use plink? Plink converts the VCF into a binary document that allows for extremely fast processing and calculation. For example, running a PCA analysis for over 6000 whole genomes in plink would take less than an hour. Doing this from a text-readable VCF would take many many hours, potentially days. So use plink!

**For an example** See `/projects/ps-gleesonlab5/user/dantaki/plink_tut/` 

## Method

* 1. Create Unique Identifiers

```
$ bcftools annotate -Oz -x ID -I '%CHROM:%POS0:%END:%REF:%ALT' myvcf.vcf.gz >myvcf.reid.vcf.gz
```

  * %CHROM : chromosome
  * %POS0  : 0-base start
  * %END   : end position
  * %REF   : reference allele
  * %ALT   : derived allele

* 2. Convert VCF into plink Bfile format

**requires plink v1.90**

```
plink --vcf myvcf.reid.vcf.gz --make-bed --out myvcf --double-id
```

  * `--make-bed`  : this will create the *.bim, *.bed, and *.fam files
  * `--out`       : output prefix
  * `--double-id` : plink assumes sample ids are formatted like **FID_IID**
    this option tells plink to write out the FID and IID as the sample ID

* 3. Update Sample information

Since plink uses binary files, we have to re-configure the family structure and genders

  * `--update-ids`
    * update the sample IDs with the correct FID and IID. `--update-ids` takes a file with this format:
    * `OLD-FID OLD-IID NEW-FID NEW-IID`

    * `plink --bfile myvcf --update-ids new.ids.txt --make-bed --out myvcf.upid`

  * `--update-parents --update-sex`
    * to calculate allele frequency and other statistics, update the parent information and sex
    * `--update-parents` takes a file with `FID IID DAD_IID MOM_IID`
    * `--update-sex` takes a file with `FID IID SEX`
      * for sex `1` is male `2` is female

  * `plink --bfile myvcf.upid --update-parents up.parents.txt --update-sex up.sex.txt --make-bed --out myvcf.final`

* 4. Count Alleles

Plink correctly determines allele frequency by omitting children. A correct estimate of allele frequency in a population only considers unique genomes. Children are half copies of their parents, thus plink only considers founders for allele frequency.

```
plink --bfile myvcf.final --freq --out myvcf
```

Get allele counts

```
plink --bfile myvcf.final --freq counts --out myvcf
```

Note that plink records the minor allele frequency, NOT the ALT allele frequency

  Examples:

  * ALT is minor allele
  * `1       1:15483:15484:G:T       T       G       0.25    4`

  * ALT is major allele
  * `1       1:28590:28591:T:TGG     T       TGG     0.25    4`

To calculate the ALT allele frequency, for variants where ALT is major, just subtract 1 from the minor allele frequency.
So for `1:28590:28591:T:TGG`, the minor allele frequency is 0.25, but the ALT allele frequency is 0.75

* 5. Find Mendelian Errors

```
plink --bfile myvcf.final --mendel --out myvcf
```

Note: this will create large files!

[.*mendel file formats](https://www.cog-genomics.org/plink/1.9/formats#mendel)


* 6. Count Transmission Rates

**use plinkv1.07** until an updated release is provided. [See this bug report](https://github.com/chrchang/plink-ng/issues/90)

Note: when running plink-1.07 supply the `--noweb` option

Before running this step, you will need to provide a phenotype file (unless you updated the phenotypes in an earlier step)

The format of this phenotype file is
  *`FID IID PHENO`
  * phenotypes in plink are `2` for cases, `1` for controls, and `0` or `-9` for missing data

**For the transmission test, plink will only test cases**

```
plink-1.07 --bfile myvcf.final --tdt --poo --pheno myvcf.pheno --out myvcf --noweb
```

The output file will be called `<outprefix.tdt.poo>`. plink will perform a transmission disequilibrium test for a given variant, which is cool for some applications, but for most analyses of transmission tests, we only care about these columns

```
# *.tdt.poo

SNP     A1:A2   T:U_PAT T:U_MAT
1:10396:10397:C:CCCCTAA CCCCTAA:C       0:0     0:0
1:10439:10440:C:CCCTAA  CCCTAA:C        1:1     1:1
1:10582:10583:G:A       A:G     1:1     0:0
1:10615:10637:CCGCCGTTGCAAAGGCGCGCCG:C  CCGCCGTTGCAAAGGCGCGCCG:C        0:0     0:0
1:12782:12783:G:A       G:A     1:1     0:0

```

`T:U` is transmitted:un-transmitted. A transmitted variant was passed from parent to child, an un-transmitted variant is present in the parent, but was not transmitted.

**Note**: it is important to check if the minor allele (`A1`) is the same as the ALT allele

---

# Tips and Tricks

precombiled plink binaries

```
# plink v1.9
/home/dantakli/bin/plink
```

```
# plink v1.07
/home/dantakli/bin/plink-1.07
```

plink will output files in an odd space-delimited manner. If you like tabs (and you should), copy over this script (or export it in your `~/.bashrc`)

```
cp /home/dantakli/bin/tabbit ~/bin/
chmod +x ~/bin/tabbit

# usage

less ssc.11002.tdt.poo | tabbit >ssc.11002.tdt.poo.txt

# or convert the file pseudo-inplace

less ssc.11002.tdt.poo | tabbit >tmp && mv tmp ssc.11002.tdt.poo

```

## Running plink

In the past, I was able to run nearly all plink commands in the login node on TSCC. But I think that day has sailed.

I would recommend either running plink on an interactive node, or to run it on Comet. Binaries compiled on TSCC will work on Comet and vice versa.

## File Formats
If you are confused about a file format, input or output, refer to this webpage

[Plink File Formats](https://www.cog-genomics.org/plink/1.9/formats)

## Common File Formats

### .fam
https://www.cog-genomics.org/plink/1.9/formats#fam

```
FAMILY_ID   INDIVIDUAL_ID   DAD_ID   MOM_ID   SEX   PHENOTYPE
```
If a sample lacks a father and/or a mother, use a `0`

Sex is encoded as `1` for males, `2` for females and `0` for unknown

Phenotype is encoded as `1` for control, `2` for case, and `0` or `-9` for unknown

For example

```
TRUMP   DONALD   0   0   1   2
TRUMP   MELANIA  0   0   2   1
TRUMP   BARRON   DONALD   MELANIA  1  1
```

Here Donald Trump is a male case with no parents, Barron Trump is the son of Donald and Melania

**NOTE** sometimes it's better to define phenotype in a separate file, like in the transmission rate section.
