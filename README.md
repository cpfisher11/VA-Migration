## Virginia is for Movers
I am still learning how to use github and am not sure why the text is so large! Maybe I have put my code in the wrong place! All advice is welcome.
ETA: Thanks, Tom! This is *much* better.

The data from 1992 year forward are in .xls format, and the tool with which I will be cleaning and analyzing my data is R, so the first thing I did was install the `gdata` package, which includes the function `read.xls.`
```
install.packages("gdata")
library(gdata)
```
Then I read in my In Migration data:
```
IN9293 <- read.xls("C9293vai.xls", header=FALSE, stringsAsFactors=FALSE)
InMig9293 <- IN9293[-(1:6),]
```
There are several irregularly formatted header lines, so I remove these, and I also tell R not to convert strings automatically to factors. (You cannot do math with columns full of factors, I learned the hard way.)

Next, I gave my columns some names, since I stripped the headings when I read the data to R.
```
dimnames(InMig9293)[[2]]<- c("In.FIPS.State", "In.FIPS.Co", "From.FIPS.State", "From.FIPS.Co", "State", 
                             "County","Returns aka Households In", "Exemptions aka Ppl In", "$")
 ```
 
…and made a new data frame containing only information about counties people moved to, not where they came from (this info is included, but not useful for my project):
```
InMigration9293 <- InMig9293[InMig9293$From.FIPS.State=="00",]
```

The IRS also includes the number of non-migrants (that is, people reporting from the same address as the previous year), so I create a new df pulling that information 
```
NonMig9293 <- InMig9293[InMig9293$County=="County Non-Migrant",]
NonMig9293$NonMigrant <- NonMig9293$`Exemptions aka Ppl In`
```
I add that info to my InMigration data frame…
```
InMigration9293$NonMigrant <- NonMig9293$NonMigrant
```
…and then create a final df containing only the columns I care about
```
FinalIn9293<- subset(InMigration9293, select=c("In.FIPS.Co", "County", "Exemptions aka Ppl In", "NonMigrant"))
```
… change the FIPS column name so I can merge it with the Out Migration data later
```
FinalIn9293$FIPS.Co <- FinalIn9293$In.FIPS.Co
```
…and update the final df to reflect that change
```
FinalIn9293<- subset(FinalIn9293, select=c("FIPS.Co", "County", 
                                           "Exemptions aka Ppl In", "NonMigrant"))
```
Hooray! On to the Out Migration data. I will spare you the play-by-play, but here’s the code:
```
Out9293 <- read.xls("C9293vao.xls", header=FALSE, stringsAsFactors=FALSE)
OutMig9293 <- Out9293[-(1:6),]
Out9293 <- gsub(",", "", Out9293)

dimnames(OutMig9293)[[2]]<- c("Out.FIPS.State", "Out.FIPS.Co", "To.FIPS.State", "To.FIPS.Co", "State", 
                              "County","Returns aka Households Out", "Exemptions aka Ppl Out", "$")

OutMigration9293 <- OutMig9293[OutMig9293$To.FIPS.State=="00",]

NonMig9293.out <- OutMig9293[OutMig9293$County=="County Non-Migrant",]
NonMig9293.out$NonMigrant <- NonMig9293.out$`Exemptions aka Ppl Out`

OutMigration9293$NonMigrant <- NonMig9293.out$NonMigrant

FinalOut9293<- subset(OutMigration9293, select=c("Out.FIPS.Co", "County", "Exemptions aka Ppl Out", "NonMigrant"))

FinalOut9293$FIPS.Co <- FinalOut9293$Out.FIPS.Co

FinalOut9293<- subset(FinalOut9293, select=c("FIPS.Co", "County", 
                                             "Exemptions aka Ppl Out", "NonMigrant"))
```

Now I want to combine all this info into a single data frame so I can calculate each county’s net migration. (Here’s where creating “FIPS.Co” pays off.)
```
NetMigration93 <- merge(FinalOut9293, FinalIn9293, by = c("FIPS.Co","County", "NonMigrant")) 
```
… get rid of some pesky commas so I can convert my non-numeric Migration numbers into numeric ones…
```
NetMigration93$MigrationIn <- as.numeric(gsub(",", "", as.character(NetMigration93$`Exemptions aka Ppl In`)))
NetMigration93$MigrationOut <- as.numeric(gsub(",", "", as.character(NetMigration93$`Exemptions aka Ppl Out`)))
NetMigration93$NonMigrant <- as.numeric(gsub(",", "", as.character(NetMigration93$NonMigrant)))
```

…which allows me to calculate the net migration:
```
NetMigration93$Net <- NetMigration93$MigrationIn - NetMigration93$MigrationOut
```
Finally, I want to calculate the percent increase or decrease relative to the 1992 total population (“population,” due to the limits I mentioned before), so I add the non-migrant totals to out-migration totals…
```
NetMigration93$Total92 <- NetMigration93$NonMigrant + NetMigration93$MigrationOut
```
…convert to a percentage…
```
NetMigration93$Percent <- (NetMigration93$Net/NetMigration93$Total92)*100
```
…and move everything into a new, complete data frame.
```
FinalNetMigration93 <- subset(NetMigration93, select = c("FIPS.Co", "County", "NonMigrant",
                                                         "MigrationIn", "MigrationOut", "Net", "Percent"))
```
Voila! 
