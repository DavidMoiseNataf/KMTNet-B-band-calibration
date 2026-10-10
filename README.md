# KMTNet B-band calibration

Photometric calibration of KMTNet $B$-band DAOPHOT catalogues of the Galactic bulge onto a fixed, KMTNet-native $B$ system
($B_{\rm K}$) defined by Gaia DR3 XP spectra. Companion code to Call, Nataf et al. (in prep.), *B-band Photometric Catalog of the
Galactic Bulge from the Korean Microlensing Telescope Network*.

## What it does

For every frame and chip (K, M, N, T) the notebook `KMTNet_B_calibration.ipynb`

1. plate-solves the catalogue with astrometry.net and refines the solution against Gaia (TAN-SIP);
2. matches the detections to Gaia DR3 (0.5", proper motions propagated to the epoch);
3. computes reference magnitudes $B_{\rm K}$ (AB) from the Gaia XP spectra of the matched stars through the system passband,
   filter transmission times the measured QE of the chip;
4. fits ${\rm MAG} - B_{\rm K} = a_0 + a_c\Delta c + a_{c^2}\Delta c^2 + a(x,y) + a_{\rm amp}$ with $\Delta c = (G_{\rm BP}-G_{\rm RP}) - 1.5$,
   five spatial terms and eight amplifier offsets, by iteratively clipped least squares;
5. writes a calibrated catalogue of every detection, with its Gaia match, $B_{\rm K}$, a predicted Johnson $B$
   (GSPC scale, via a colour transformation fitted once between the two synthetic systems) and a diagnostic figure.

Later sections summarise the program, compare repeated pointings and the master flat, and produce the figures and tables of the
paper.

## Inputs

- `xkmts.YYYYMMDD.NNNNNN_combined.csv` — four-chip DAOPHOT catalogues (`CHIP, ORIG_ID, X, Y, RA, DEC, MAG, MERR, MSKY, NITER, CHI, SHARP`)
- `xkmts.YYYYMMDD.NNNNNN.fits` — the frames, read for their headers only (airmass, date, object); or a `frame_meta.csv` with
  columns `frame, airmass, date_obs, exptime, object`
- `FilterTransmission.txt` — lab transmission of the filter (nm, then T% at nine positions)
- optional: `flatB.fits` (master flat, extensions named `K01`…`T08`), the CALSPEC Vega spectrum, `anet_api_key.txt`

Gaia DR3 sources and XP spectra are taken from the ESA bulk files (downloaded on first use and kept), the GSPC Johnson $B$ used for
the Johnson transformation is generated locally from those files with GaiaXPy (`gspc_source = "archive"` queries the Gaia archive
instead), and the Roman GBTDS field layout comes from [mtpenny/gbtds_optimizer](https://github.com/mtpenny/gbtds_optimizer).

## Outputs

`<stem>_calibrated_catalog.csv/.fits` and `<stem>_calibration.json` per chip-frame, `calibration_summary.csv`,
`coefficients_table.tex`, `figs/`. Catalogue columns: the DAOPHOT measurements, `B_K`, `B_K_err`, `B_J_pred`,
the colour used and its source (`bp_rp`, `B-G`, `assumed`), `matched`, `sep_arcsec`, `calib_star`, `B_K_ref` (XP stars only), and the
Gaia DR3 source parameters of the match (empty if none). Read the csv with `dtype={'source_id': 'Int64'}`, or use the FITS, to keep the
19-digit identifiers exact.

## Running

Python ≥ 3.9 with `numpy, scipy, pandas, matplotlib, astropy, astroquery, astropy_healpix, gaiaxpy, requests`. Edit the
configuration block at the top of the notebook (frame glob, chips, site) and run all cells. The batch is resumable and skips
chip-frames that already have a record from the current notebook version; the first pass is dominated by downloads
(tens of GB of Gaia bulk files, shared by all fields), afterwards a chip-frame takes about a minute.

## Running on another machine

Nothing in the code is tied to this computer, but these are specific to the program and the setup:

- the data files listed under Inputs, in the working directory (or edit `frames_glob`, `frame_meta_csv`, `filter_file`);
- the configuration block: `chips`, `site` (chip QE), `skip_items`, and in Section 8 `paper_frame`, `paper_chip`, `paper_cmd_items`,
  which name frames of this program; the QE tables in Section 1 are for the KMTNet CCDs;
- an astrometry.net API key in `anet_api_key.txt` (or `$ASTROMETRY_NET_API_KEY`) when catalogues still need plate solving;
- internet access to the ESA Gaia CDN, GitHub, CDS (hips2fits) and nova.astrometry.net, and tens of GB of disk for the Gaia and XP
  bulk files (`gaia_bulk_files/`, `xp_bulk_files/`);
- time: with the caches in place a chip-frame takes 5–25 s (the full program under an hour); from scratch, the astrometry.net solves
  and the downloads dominate.

## License and acknowledgments

MIT License (see `LICENSE`). The notebook downloads, but does not redistribute, the Roman GBTDS field layouts of
[mtpenny/gbtds_optimizer](https://github.com/mtpenny/gbtds_optimizer) and the Gaia DR3 colour flux map served by the CDS
hips2fits service; if you use the sky-image version of the coverage figure, acknowledge hips2fits (CDS) as requested at
https://alasky.cds.unistra.fr/hips-image-services/hips2fits. GSPC-scale Johnson $B$ is generated with GaiaXPy.
