This folder is where you put the vendor reader libraries that NovoCRF uses to read Thermo .raw and Bruker
timsTOF .d files. It starts empty. You download the libraries from the vendor and put them here; the exact
sub-folder and version do not matter, NovoCRF searches this folder by file name.

Thermo .raw:
  Download RawFileReader from https://github.com/thermofisherlsms/RawFileReader
  Put these two files anywhere under vendor_reader/ (e.g. vendor_reader/thermo/):
    ThermoFisher.CommonCore.RawFileReader.dll
    ThermoFisher.CommonCore.Data.dll
  Take them from Libs/NetCore/Net8 (lib/net8.0/ inside the .nupkg files), the same files for Windows and Linux.
  The DLLs in Libs/Net48 have the same names but are not the build NovoCRF uses.

Bruker timsTOF .d:
  Download the TDF SDK from
  https://www.bruker.com/service/support-upgrades/software-downloads/mass-spectrometry.html
  Put the shared library for your platform anywhere under vendor_reader/ (e.g. vendor_reader/bruker/):
    Windows: timsdata.dll        Linux: libtimsdata.so

These files are provided by the vendor under the vendor's own licence, which you accept when you download them.
They are not supplied, supported or endorsed by NovoCRF or by the vendor as part of NovoCRF. You do not need them
for MGF input.
