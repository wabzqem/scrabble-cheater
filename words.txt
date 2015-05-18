#!/usr/bin/perl -w

# Copyright 2007 Richard Nelson

use strict;
use CGI qw(:standard);

open FH, "TWL06.txt";
my @all = <FH>;
close FH;

my %perms;

sub getreg { #builds a regex like (?=.*O.*O)(?=.*P)(?=.*L) given OPOL, for example
	my @m = @{$_[0]};
	my %cl;
	$cl{$_}++ foreach @m; #count how many of each letter
	my %saw;
	@saw{@m} = (); #unique m
	@m = keys %saw;
	my @match;
	foreach my $l (@m) { #build (?=.*O.O) for each letter
		my $n = "(?=";
		for (my $i = 0; $i < $cl{$l}; $i++) {
			$n .= ".*$l";
		}
		$n .= ')';
		push @match, $n;
	}
	my $match = join '', @match;
	$match;
}

sub remove_nth {
	my @c = @{$_[0]};
	my $i = $_[1];
	my @r;
	for (my $j = 0; $j < @c; $j++) {
		push @r, $c[$j] if $j != $i-1;
	}
	\@r;
}

sub recur {
	my @m = @{$_[0]};
	return if (@m == 0);
	my $perm = join('', sort @m);
	$perms{$perm}++;

	for (my $l = @m; $l > 0; $l--) {
		my @current = @{remove_nth(\@m, $l)};
		recur(\@current);
	}
}
sub remove_one {
	my @copy;
	my @m = @{$_[0]};
	my $l = $_[1];
	my $f = 0;
	foreach (@m) {
		if ((not $f) && ($_ eq $l)) {
			$f = 1;
		} else {
			push @copy, $_;
		}
	}
	\@copy;
}
my $debug = param('debug');
my $input = uc(param('l'));
my $c = param('n');
my $start = uc param('s') || "";
my $end = uc param('e') || "";
print "Content-Type: text/plain\n\n";
unless ($input && (length($input) < 9) && defined($c) && $input =~ /^[A-z]+$/ && $c =~ /^[0-9]+$/ && $c < 10)  {
	print "No...\n";
	return;
}
my @m = split //, $input;
my $count = scalar @m;
my $match = getreg(\@m);
#@m = grep { $_ ne $start } @m;
#@m = grep { $_ ne $end } @m;
@m = @{remove_one(\@m, $start)};
@m = @{remove_one(\@m, $end)};
if ($c && $c >= $count && (!($c == $count && ($start xor $end)))) {
	if ($end && $start) {
		$c = $c - 2;
	} elsif ($end || $start) {
		$c = $c - 1;
	}
	print "MATCHING: $match^$start.{$c}$end.\$\n" if $debug;
	my @words = grep(/$match^$start.{$c}$end.$/, @all);
	print foreach (@words);
} else {
	recur(\@m);
	if ($debug) {
		print "'$_'\n" foreach (keys %perms);
	}
	foreach my $letters (keys %perms) {
		my @l = split '', $letters;
		my $z = @l;
		if ($end && $start) {
			next if ($c && $c != @l + 2);
		} elsif ($start || $end) {
			next if ($c && $c != @l + 1);
		} else {
			next if ($c && $c != @l);
		}
		$match = getreg(\@l);
		my @words = grep(/^$start$match.{$z}$end.$/, @all);
		print "MATCHING: ^$start$match.{$z}$end.\$\n" if $debug;
		print foreach (@words);
	}
}
