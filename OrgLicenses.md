# Ontology of Licenses

## Objective

Organize different types of open source licenses into an ontology of
semantically similar licenses.  The input data would be all license
files that exist, the first step would be do do an unsupervised
clustering and then labeling using known types of licenses and,
perhaps, developing a supervised (more accurate) technique to
determine license type from its text.



```
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