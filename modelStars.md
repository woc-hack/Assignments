### Note
files prj, prjSU0, etc used below are in /home/audris, so you do not need to regenerate them

### First get project summaries from P_metadata.T: only projects with 10+ days from first to last commit are selected
```bash
python3 listPM.py > prj
#get stars from latest ghtorrent dump
zcat /da5_data/basemaps/gz/ght.P2w.cnt | lsort 10G -u -t\; -k1,1 | join -t\; -a1 <(lsort 1G -t\; -k1,1 prj) - > prjS
#identify gh repos under individual users (not organizations): again use latest ghtorrent dump
zcat /da5_data/basemaps/gz/ght.users.csv1|cut -d\, -f2,5|grep ,USR | cut -d, -f1 |lsort 1G -u |grep -v _ > prjU
# merge both and prepare for R input
lsort 1G -t\; -k1,1 <(grep _ prjS|sed 's|_|;|') |join -t\; <(cat prjU | grep -v _ | lsort 1G -t\; -k1,1) - | sed 's|;|_|' > prjSU
awk -F\; '{if(NF==7){print $0";0"}else{print $0;}}' prjSU > prjSU0
```
### Now use R for sampling
```R
x  = read.table("prjSU0",sep=";",quote="",comment.char="")
#name fields (nc1 is commits by top developer)
names(x) = c("p","dur","fr","na","nc","nCore","nc1","ns")
#calculate logs and ranks
x$lns=log(x$ns+1);x$ldur=log(x$dur+1);x$lna=log(x$na+1);x$lnc=log(x$nc+1);x$rat=x$nc1/x$nc;
checkc = quantile(x$lnc,(0:100)/100);checka = quantile(x$lna,(0:100)/100);checkd = quantile(x$ldur,(0:100)/100);checks = quantile(x$lns,(0:100)/100);checkr = quantile(x$rat,(0:100)/100);
# sometimes log-transform still leaves outliers with such extremely skewed data
# Using rank-transform makes the analysis more robust to such outliers
for (f in c("lns","lnc","lna","ldur","rat")){
  f1=paste(f,"r",sep="");
  x[,f1] = rank(x[,f]);
}
# get 100 quantiles for matching unrelated repos
checkc = quantile(x$lnc,(0:100)/100);checka = quantile(x$lna,(0:100)/100);checkd = quantile(x$ldur,(0:100)/100);checks = quantile(x$lns,(0:100)/100);checkr = quantile(x$rat,(0:100)/100);

#randomly sample 10M repos
sel = sample(1:dim(x)[1],10000000)
#first a smaller sample for analysis then a larger sample for matching
za=x[sel,][1:1e6,]    # 1M
zb=x[sel,][1e6+1:10e6,] #9M

#get the number of stars from unrelated but matched by commits or duration repositories
for (j in 1:100){ sa = za$lnc>= checkc[j] & za$lnc < checkc[j+1];lsa = sum(sa); za$lnsc[sa] = zb$lns[zb$lnc>= checkc[j] & zb$lnc < checkc[j+1]][1:lsa]; };
for (j in 1:100){ sa = za$ldur>= checkd[j] & za$ldur < checkd[j+1];lsa = sum(sa); za$lnsd[sa] = zb$lns[zb$ldur>= checkd[j] & zb$ldur < checkd[j+1]][1:lsa]; };

#Lets take a look at residual from simple models 
mml=lm(lns~ldur+lnc+lna+rat, data=za,subs=za$na>0);
mm=gam(lns~ldur+lnc+lna+s(rat), data=za,subs=za$na>0);
png("res.out",width=2000,height=2000)
par(mfrow=c(2,3))
plot(mm)
plot(lowess(za$rat[za$na>0],mm$residuals))
plot(lowess(za$rat[za$na>0],mml$residuals))
dev.off()
# a really odd regular shape: why?
mml300=lm(lns~ldur+lnc+lna+rat, data=za,subs=za$na>300);
mm300=gam(lns~ldur+lnc+lna+s(rat), data=za,subs=za$na>300);
png("res300.out",width=2000,height=2000)
par(mfrow=c(2,3))
plot(mm300)
plot(lowess(za$rat[za$na>300],mm300$residuals))
plot(lowess(za$rat[za$na>300],mml300$residuals))
dev.off()
# a really odd regular but different from above shape: why?

#projects with 9+ authors seem to offer good fit
summary(lm(lns~ldur+lnc+lna+rat, data=za,subs=za$na>8))
# what is rat: nc1/nc, but nc is already in the model, this is an even better fit!
summary(lm(lns~ldur+lnc+lna+log(nc1), data=za,subs=za$na>8))
# the two models were, in fact, equivalent if log(rat) was used instead: why? 
# does the interpretation differ, if so, why?

#What about replacing with stars from unrelated projects
summary(lm(lns~ldur+lnc+lna+rat, data=za,subs=za$na>0))
summary(lm(lnsd~ldur+lnc+lna+rat, data=za,subs=za$na>0))
# seems like the relationship is still there
# What could that mean: we fond the same relationship when we get the response variable from totally unrelated projects?

#now get the info on the first 10 commits
write(as.character(za[,1]),file="pza",ncol=1)
```
### calculate relevant properties
```bash
#The sample of projects in pza, lets get commits sorted by time
cat pza | while read i; do echo $i; done | ~/lookup/getValues -f P2c | awk -F\; '{print $2";"$1}' | ~/lookup/getValues c2dat | perl -ane 's|^[^;]+;||;@x=split(/;/); print "$x[0];$x[1];$x[3]\n"'  | lsort 100G -t\; -k1,2 | gzip > pza.byTime
#now charaterize the first 10 commits for each project
zcat pza.byTime| perl -e 'while(<STDIN>){chop();($p,$t,$a)=split(/;/);next if $c{$p}>=10;$c{$p}++;$d{$p}{$a}++;$tf{$p}=$t if $c{$p}==1;$tl{$p}=$t if $c{$p}==10}for $p (keys %d){@x=sort { $d{$p}{$b} <=> $d{$p}{$a} } (keys %{$d{$p}});print "$p;$x[0];$d{$p}{$x[0]};$c{$p};$tf{$p};$tl{$p};$#x\n"}' | gzip > pza.f10
```
### Get back to analysis in R
```R
zaf = read.table("pza.f10",sep=";",quote="",comment.char="")
mat = match(za$p,zaf[,1])
# number of commits by top person from among the first 10 commits (f - first)
za$nf=zaf[mat,3]
# duration until tenth commit
za$d10=zaf[mat,6]-zaf[mat,5]
# number of distinct authors for these 10 commits
za$na10=zaf[mat,7]+1

#lets model stars based on number of initial 10 commits by top developer and the time until 10th commit
summary(lm(lns~log(d10+1)+I(as.factor(nf)), data=za,subs=za$na>0))
# Seems that only duration matters, not other predictors. 
# now select only projects that had more than 1 authors eventually
summary(lm(lns~log(d10+1)+I(as.factor(nf)), data=za,subs=za$na>1))
# looks like more levels of nf matter: why?
#now select only projects that had more than 10 authors eventually
summary(lm(lns~log(d10+1)+I(as.factor(nf)), data=za,subs=za$na>10))
# looks like even more levels of nf matter: why?
```


### Below is python code to get project summaries
```python
import sys, re, pymongo, json
import requests

client = pymongo.MongoClient ('da1')
# Get a reference to a particular database
db = client ['WoC']
# Reference a particular collection in the database
coll = db ['P_metadata.T']
query = json.loads('{}')


fl = ["NumAuthors", "NumCommits", "LatestCommitDate", "EarlistCommitDate", "Core"]
for r in coll .find (query):
  skip = 0
  for f in fl:
    if f not in r: skip = 1
  if skip == 0:
    dur = r["LatestCommitDate"] - r["EarlistCommitDate"]
    ncmt = int (r["NumCommits"])
    # get only 10+ day projects measured va last - first commit dates
    if (ncmt > 10 and dur/3600/24 > 10):
      cr = r["Core"]
      mx = 0;
      # pick larges core contributor
      for a in cr:
        if int(cr[a]) > mx: mx = int(cr[a])
      print (r["ProjectID"] + ";" + str(dur)+";"+str(r["EarlistCommitDate"])+";"+str(r["NumAuthors"])+";"+str(r["NumCommits"])+";"+str(r["NumCore"])+";"+str(mx))
```
