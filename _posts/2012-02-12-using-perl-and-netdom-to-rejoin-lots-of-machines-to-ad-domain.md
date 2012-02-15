---
layout: post
title: "Using perl and netdom to rejoin lots of machines to AD domain"
category: ["perl"]
tags: ["perl", "LDAP", "netdom"]
date: 2012-02-12 16:41:00
---
{% include JB/setup %}

A few years back we had an incident where somehow the computers OU got deleted from the domain. Someone from our systems team was able to restore the OU from what I can only assume was an LDIF backup. However, although the machines may have been listed in the OU when any given user attempted to log in to thier computer they received an error stating that the machine was no longer on the domain. I wrote this script in order to do a massive rejoin of all computer objects from a list of machines from that OU. 

*rejoin.pl*
<pre class="prettyprint linenums">
use Net::Ping;
open (COMPS, 'thelist.txt');
my @computers =<COMPS>;
foreach(@computers) {
    chomp(); if(/^#/) { next } pingit($_) };
sub pingit(){
 my $p = Net::Ping->new();
 #domainUser must be able to join machines to the domain and be in the Administrator's Group on the machine
 system("netdom remove $_ /domain:example.tld /userd:domUser /passwordd:domUserPass /usero:LocalAdmin /passwordo:LocalAdminPass /verbose") if $p->ping($_);
 system("netdom join $_ /domain:example.tld /userd:domUser /passwordd:domUserPass /usero:LocalAdmin /passwordo:LocalAdminPass /reboot /verbose") if $p->ping($_);
 system("echo $_ >> computers.txt") if $p->ping($_);}	
</pre> 

It took me a few hours trying to get the syntax correct on the netdom command. Mainly because if a user had attempted to log in to a machine prior to the LDAP OU being restored from LDIF that machine would have lost its trust relationship with the domain and would require that the machine in question be manually rejoined to the network. Luckily there were only a few of those machines, but as most of the machines that I was working on when trying to get the correct netdom syntax were having a trust relationship error I ending up giving up. After I started to join several machines back by hand I noticed that most of them did not have that issue and I went back to attempting to get the correct netdom syntax. All in all it had about an 87% success rate accross the campus. Most of the machines that it failed on were because they were not turned on or were on the wireless network or off campus. 

In order to speed the process of getting all of the machines back on the domain we also had provided the help desk with a copy of the script that would join one machine at a time rather that the whole list of machines. It was a simple modification which I will not include here as if you have any knowledge of perl it should be obvious as to what changes need to be made. As a consquence of providing the script to the help desk it became nessicary to change the local admin password on all the machines as that was not information that help desk should generally have had. So I wrote another script, actually a combination of several scripts, to get the password changed for the local admin account on each of the staff computers. 

*README*
<pre class="prettyprint linenums">
perl setup.pl
It will ask for your password its is your standard domain password. 
runme.bat
its done.
</pre>

*FindMyDN.vbs*
<pre class="prettyprint linenums">
Set objADSysInfo = CreateObject("ADSystemInfo")
wscript.echo objADSysInfo.UserName
</pre>

*passchange.vbs*
<pre class="prettyprint linenums">
strComputer = WScript.Arguments.Item(0)

Set objUser = GetObject("WinNT://" & strComputer & "/LocalAdmin")
objUser.SetPassword("NEWPASSWORD")
</pre>

*passchange.pl*
<pre class="prettyprint linenums">
use Net::Ping;
use Cwd;
use File::Basename;
$dir = fastgetcwd;
$tld = fileparse("$dir");

system("echo \#Not Done > notdone.txt");
open (COMPS, "$tld.txt");
my @computers =<COMPS>;
foreach(@computers) {
    chomp(); if(/^#/) { next } pingitbitch($_) };
sub pingitbitch(){
 my $p = Net::Ping->new();
 #system("cscript passchange.vbs $_") if $p->ping($_); #change password 
 my $re='\(null\): (.*)\n';
 my @outp = `cscript passchange.vbs $_ 2>&1` if $p->ping($_);
 my $error;
 foreach my $item (@outp) {
      if($item =~ m/$re/is ) { $error = $1; last; }
    }
 system("echo $_ $error  >> error.txt") if $error;
 system("echo $_ >> done.txt") if $p->ping($_);	#appears to be successful
 system("echo $_ >> notdone.txt") unless $p->ping($_);} #there was a problem here
close (COMPS);
system("cp notdone.txt $tld.txt");
</pre>

*setup.pl*
<pre class="prettyprint linenums">
#passchange2.pl
# This script was written with the intent to allow for easier password changes for the local admin account
#
# If you have questions reguarding this script please contact JD Geier in the ITS department via email: 
# jgeier@regis.edu or via my work extention @ x5090
#
# This script was writen for Regis University and may not work elsewhere without modification.
# (c) 2009 Regis University, Jd Geier.

use Net::LDAP;
use Net::LDAP::Entry;
use Getopt::Std;
use Term::ReadKey;
use vars qw/ %opt /;
use strict;
use warnings;

my $mesg;
my $ldap;
my $username = `cscript /NoLogo FindMyDN.vbs`; $username =~ s/\cM\cJ//;
my $password;
my $base;
my $port = 389;
my $scheme = 'ldap';
my $ip;
my $version = 3;
my $scope = 'one';
my $filter = "(objectClass=organizationalUnit)";

sub init()
{
 my $opt_string = 'hp:b';
 getopts( "$opt_string", \%opt ) or usage();
 usage() if defined $opt{h};
}

#
# Message about this program and how to use it
#

sub usage()
{
    print STDERR << "EOF";
This program uses Net::LDAP to get a list of computers in the Computers OU. After it gets the list of computers it will proceed to change password logs then placed into a CSV file located in the same place the script is run from. This script only works with the user that is logged in. If you wish to run the application as yourself please log the current user out and login using your own credentials. This is a limitation of the script due to the compexities of the CNs for AD.


usage: $0 [-h] [-p Password] [-b OU] [-a] 
 -h        : Displays this help message.
 -p        : Password. If password is not defined the application will ask for you to enter it.
 -b        : Run in specified OU
 -a		   : All OUs

example: $0 [-h] [-p password] 
EOF
    exit;
}

init();

sub getPassWord()
{
 print "Please enter your password: \n";
 ReadMode('noecho');
 chomp(my $passwordIn = <STDIN>);
 ReadMode(0);
 return $passwordIn;
}

$password =       $opt{p}        if     defined $opt{p};
$password = getPassWord()        unless defined $opt{p};

$ip = '255.255.255.255'; #replace with your IP address of the LDAP server
 $ldap = Net::LDAP->new( 'ldapserverdomainname') or die('.Could not connect to LDAP server..');


 $mesg = $ldap->    bind (               $username, 
		            password          => $password);		

$base= "OU=Computers,DC=exmaple,DC=tld"; #replace with the base ou that your computers are stored in. 

print "$base \n";
$mesg = $ldap->search( base => $base, 
	
					    filter => $filter);
						$mesg->code && die $mesg->error;
my $max = $mesg->count;
my @ous;
for( my $i = 0 ; $i < $max ; $i++) {
 my $entry = $mesg->entry ($i);
 my @attrs = $entry->attributes;
 my $ouname;
 
 foreach my $attr (@attrs) {
 	if ($attr eq "name") { 
 		$ouname = $entry->get_value($attr);
 		push (@ous, $ouname) # unless $ouname eq "Computers"; you may need to use this as a filter uncomment and modify as needed
 	}
  }
}

foreach my $ou (@ous) {
	system("mkdir \"$ou\"");
	my @computers;
	$base = "OU=$ou,OU=Computers,DC=example,DC=tld"; #If you have multiple OUs
	$filter = my $filter = "(objectClass=computer)";
	$mesg = $ldap->search( base => $base, 
	
					    filter => $filter);
						$mesg->code && die $mesg->error;
	my $max = $mesg->count;
	for( my $i = 0 ; $i < $max ; $i++) {
		 my $entry = $mesg->entry ($i);
 		 my @attrs = $entry->attributes;
 		 my $compname;
 
 		foreach my $attr (@attrs) {
 			if ($attr eq "name") { 
 				$compname = $entry->get_value($attr);
 				push (@computers, $compname );
 			}
 	  	  }
		}
	print "$base \n";
	open FILE, ">", ".\\$ou\\$ou.txt" or die $!;
	print FILE join "\n", @computers;
	system("cp passchange\.pl .\\$ou\\");
	system("cp passchange\.vbs .\\$ou\\");
	system("echo cd \.\\$ou >> runme.bat");
	system("echo perl passchange.pl >> runme.bat");
	close FILE;
}

system("echo cd .. >> runme.bat");
</pre>