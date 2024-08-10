---
layout: post
title: "parsing file by token"
description: "parsing file by token"
date: 2024-08-05
tags: [compilation, tokenizer]
---


```perl
sub parse_file($)
{
    open my $in, $yfile
        or die "Cannot open file $yfile for reaing: $!\n";
    my $s = do { local $/; <$in> };
    close $in;

    while (1) {
        # skip single quote string
        if ($s =~ /\G\s*('(?:\\.|[^'\\]+)*')/gcsm) {
            next;

        # skip double quote string
        } elsif ($s =~ /\G\s*("(?:\\.|[^"\\]+)*")/gcsm) {
            next;

        # skip comment like /* xxx */
        } elsif ($s =~ m{\G\s*(/\*.*?\*/)}gcsm) {
            next;

        # skip comment like // xxx
        } elsif ($s =~ m{\G\s*//[^\n]*\n?}gcms) {
            next;

        } elsif ($s =~ m{\G\s*([^'"/]+)}gcsm) {
            my $seg = $1;
            while ($seg =~ /\b([_a-zA-Z]\w*)/g) {
                my $name = $1;
                print($name);
            }
        # match single / with optional space
        } elsif ($s =~ /\G\s*(.)/gcsm) {
            next;

        } else {
            last;
        }
    }
}
```
