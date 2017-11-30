#!/usr/bin/env python

from __future__ import print_function

import sys
import re


def main(dirname):
    # Expecting a directory name like sometext---dc.identifier---accession
    collection_id = re.match('M\d+', dirname)
    try:
        print('"' + collection_id.group(0) + '"')  # Accession ID must be quoted
    except Exception:
        print('None')


if __name__ == '__main__':
    main(sys.argv[1])
    sys.exit(0)
