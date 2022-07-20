### Track covariance issuee fix

The credits of this goes to Tony and Olmo. To cope with the not positive defined covariance matrix errors found in miniAODs, The fix is to be applied on the recotracks that you want to use in the vertexing.

```C
reco::Track fix_track(const reco::Track *tk, double delta)
{
unsigned int i, j;
double min_eig = 1; // Get the original covariance matrix. reco::TrackBase::CovarianceMatrix cov = tk->covariance();
// Convert it from an SMatrix to a TMatrixD so we can get the eigenvalues. TMatrixDSym new_cov(cov.kRows);
for (i = 0; i < cov.kRows; i++) {
        for (j = 0; j < cov.kRows; j++) {
            // Need to check for nan or inf, because for some reason these cause a segfault when calling Eigenvectors()
            if (std::isnan(cov(i,j)) || std::isinf(cov(i,j)))
cov(i,j) = 1e-6; new_cov(i,j) = cov(i,j);
        }
    } // next - Get the eigenvalues.
    TVectorD eig(cov.kRows);
    new_cov.EigenVectors(eig);
    for (i = 0; i < cov.kRows; i++)
        if (eig(i) < min_eig)
            min_eig = eig(i);
// If the minimum eigenvalue is less than zero, then subtract it from the diagonal and add `delta`. if (min_eig < 0) {
        for (i = 0; i < cov.kRows; i++)
            cov(i,i) -= min_eig - delta;
}
return reco::Track(tk->chi2(), tk->ndof(), tk->referencePoint(), tk->momentum(), tk->charge(), cov, tk->algo(), (reco::TrackBase::TrackQuality) tk->qualityMask()); 
}

```

with delta that may be set by default to `1e-8`.  
