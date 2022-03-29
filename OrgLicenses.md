# Ontology of Licenses

## Objective

Organize different types of open source licenses into an ontology of
semantically similar licenses.  The input data would be all license
files that exist, the first step would be do do an unsupervised
clustering and then labeling using known types of licenses and,
perhaps, developing a supervised (more accurate) technique to
determine license type from its text.

```
# Get all blobs corresponding to license files
ver=T
for l in {0..127}
do zcat b2fLICENSEFull$ver$l.s | \
   grep -E ';(LICENSE|LICENSE.txt|LICENSE.md)$' | \
   cut -d\; -f1 | uniq | \
   join -t\; - <(zcat b2P128Full$ver.$l.gz) | gzip > bL2PFull$ver$l.s
   echo bL2PFull$ver$l.s
done

# Find licences with most projects
for l in {0..127}
do zcat bL2PFull$ver$l.s | cut -d\; -f1 | uniq -c |\
   sed 's|^\s*||;s| |;|' | \
   perl -ane 'chop();($n,$l)=split(/;/); print "$l;$n\n"'
done | gzip > bL2nPFull$ver.s
echo bL2nPFull$ver.s

# Find projects with most licenses

for l in {0..127}
do zcat bL2PFull$ver$l.s|perl -ane 'chop();($n,$l)=split(/;/); print "$l;$n\n"' 
done | perl -I $HOME/lib/perl5 -I $HOME/lookup $HOME/lookup/splitSecCh.perl  P2LFull$ver. 32
echo P2LFull$ver.

for i in {0..31}
do zcat P2LFull$ver.$i.gz|$HOME/bin/lsort ${maxM}M -t\; -k1,2 -u | gzip > P2LFull$ver$i.s
   echo P2LFull$ver$i.s
done
```

## Cluster licenses

Use a text analysis technique to cluster license file text and, perhaps, label clusters based on the simlarity to official licenses: see below. 


## Discover license repositories for labeling licenses

- An official list of licenses: https://opensource.org/licenses/category
- Analysis of license compatibility: https://www.whitesourcesoftware.com/resources/blog/license-compatibility/

Use the above with some text analysis technique 

## Extract licenses from source code files

Identify and classify comments at the beginning of the file as license text
- comments are language specific
- search for specific keywords


## References on empirical work related to OSS licenses and their compatibility

- Wu, Y., Manabe, Y., Kanda, T., German, D. M., & Inoue, K. (2015, May). A method to detect license inconsistencies in large-scale open source projects. In 2015 IEEE/ACM 12th Working Conference on Mining Software Repositories (pp. 324-333). IEEE.
- Wu, Y., Manabe, Y., Kanda, T., German, D. M., & Inoue, K. (2017). Analysis of license inconsistency in large collections of open source projects. Empirical Software Engineering, 22(3), 1194-1222.
- Qiu, S., German, D. M., & Inoue, K. (2021). Empirical Study on Dependency-related License Violation in the JavaScript Package Ecosystem. Journal of Information Processing, 29, 296-304.  https://www.jstage.jst.go.jp/article/ipsjjip/29/0/29_296/_pdf
- Vendome, C., Linares-Vásquez, M., Bavota, G., Di Penta, M., German, D., & Poshyvanyk, D. (2017, May). Machine learning-based detection of open source license exceptions. In 2017 IEEE/ACM 39th International Conference on Software Engineering (ICSE) (pp. 118-129). IEEE.


