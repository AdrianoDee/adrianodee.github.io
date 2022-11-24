## Tracking for BPH

Here I'm collecting some pointers, instructions, suggestions about tracking that could be useful for BPH analysis. :
1. [HLT Filter Informations in AODs](https://github.com/AdrianoDee/adrianodee.github.io/edit/main/Track4BPH/Tracking%20For%20BPH.md#hlt-filter-informations-in-aods)
2. [Tracks From a Vertex in MiniAODs](https://github.com/AdrianoDee/adrianodee.github.io/edit/main/Track4BPH/Tracking%20For%20BPH.md#tracks-from-a-vertex-in-miniaods)
3. [Track covariance issuee fix](https://github.com/AdrianoDee/adrianodee.github.io/edit/main/Track4BPH/Tracking%20For%20BPH.md#track-covariance-issuee-fix)
4. [Misalignment Systematics](https://github.com/AdrianoDee/adrianodee.github.io/edit/main/Track4BPH/Tracking%20For%20BPH.md#misalignment-systematics)

-------
### HLT Filter Informations in AODs

You need your analyzer to consume a `trigger::TriggerEvent>` object:

```cpp
 edm::Handle<trigger::TriggerEvent> triggerEvent;
  iEvent.getByToken(TriggerEventToken_, triggerEvent);
  
```
the `InputTag` should be `hltTriggerSummaryAOD` (check in your file anyway):

```python
TriggerEvent = cms.InputTag("hltTriggerSummaryAOD"),
```

and most probably you would want to define a list of filters you are interested in `TriggerFilters = cms.vstring(["hltDoubleMu2JpsiDoubleTrkL3Filtered"])`.

Then you can access the objects that fired a specific set of filters with something like

```cpp
  auto processName = triggerEvent->usedProcessName();
  std::cout << "----> Input Filters" << std::endl;
  for (size_t i = 0; i < TriggerFilters_.size(); i++) {
    std::cout << "> " << TriggerFilters_[i] << std::endl;
    const unsigned int filterIndex(triggerEvent->filterIndex(edm::InputTag(TriggerFilters_[i], "", processName)));

    if (filterIndex < triggerEvent->sizeFilters()) { // this tells us if the filter has been fired
      const trigger::Keys& keys(triggerEvent->filterKeys(filterIndex));
      const trigger::Vids& ids(triggerEvent->filterIds(filterIndex));
      const size_t nObjects(keys.size());

      for (size_t i = 0; i < nObjects; i++) {
        const trigger::TriggerObject& tObject(triggerObjects[keys[i]]);
        std::cout << "   " << i << " " << ids[i] << "/" << keys[i] << ": " << tObject.id() << " "
                                             << tObject.pt() << " " << tObject.eta() << " " << tObject.phi() << " " << tObject.mass() << endl;

      }
    }
  }
```

Then you build the best criteria that you want to match these objects to your RECO or PAT objects. See the example [here](https://github.com/AdrianoDee/tracksAnalyzers/tree/main/TrackAnalyzer/TrackAnalyzer/src).

-------
### Tracks From a Vertex in MiniAODs

The vertex association of a packed candidate may be checked with two methods:

- pvAssociationQuality() returns the association quality w.r.t to the vertex associated to the track (the one you get with vertexRef())
- fromPV() is actually fromPV(size_t ipv=0) so it return “association” of the vertex w.r.t to the vertex with index==ipv. So when you use it as it is will return the flag (0,1,2 or 3) w.r.t to the PV at position 0, i.e. the leading PV. If you want to check the association w.r.t. to the vertex associate to the candidate you need to get the vertexRef().key() as ipv.

__Note__ that not all the tracks participate in the building of a PV and, more, not all the PF candidates are finally associated to a vertex. So in general the flow is:

1. reconstruct the PVs from tracks;
2. build the PF candidates (some of them having a track);
3. associate the PF candidate to a vertex (from here you get different qualities);
4. if the PF is not associated to any vertex the vertex ref is simply the PV __nearer in dz__. For this case the association quality will be 0 (or anyway low).

You can “see” the flow [here](https://cmssdt.cern.ch/dxr/CMSSW/source/PhysicsTools/PatAlgos/plugins/PATPackedCandidateProducer.cc?from=PATPackedCandidateProducer&case=true#245). That said one may check select only tracks coming from a specific vertex (with different degrees of association) using the `vertexRef()` and check if corresponds to the vertex you want using the vertex `key()`.

```cpp

...
//input collections
edm::Handle<edm::View<pat::PackedCandidate> > cands;
iEvent.getByToken(CandsCollection_,cands);

edm::Handle<reco::VertexCollection> primaryVertices_handle;
iEvent.getByToken(thePVs_, primaryVertices_handle);

...
//You select a vertex with your own criteria and get THEKEY of its edm::Ref
reco::VertexRef vtx(primaryVertices_handle, THEKEY);
...
  
 for (unsigned int i=0; i< cands->size(); i++)
    {
    //when you loop on the 
    pat::PackedCandidate cand = cands->at(i);
    auto thisVertex = cand.vertexRef();
    
   if(thisVertex.key()!=THEKEY)
      continue
   }
...

```
-------
### Track covariance issuee fix

The credits of this goes to Tony and Olmo. To cope with the not positive defined covariance matrix errors found in miniAODs, The fix is to be applied on the recotracks that you want to use in the vertexing. The idea is to subtract the minimum eigenvalue from the diagonal if that is negative and to add a positive delta to it (could be ~0).

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

with delta that may be set by default to `1e-8`. So using this as:

```C
auto fixedTrack = fix_track(candidate->bestTrack());
```
the `fixedTrack` will have the covariance defined positive.

-------
### Misalignment Systematics

Thanks to Sara for these instructions and documentation.

To estimate the systematics due to tracker misalignment have a look at an example here:

https://cms.cern.ch/iCMS/jsp/db_notes/noteInfo.jsp?cmsnoteid=CMS%20AN-2015/110
(in sec. 3.8.5)

This was Run1, and follows the procedure described here

https://twiki.cern.ch/twiki/bin/view/CMS/SystematicMisalignmentsofTracker

Which is still valid. The idea is to 

- have new GTs with alignments different from the standard ones.
- re run the MC reco with new tags (that includes misalign)
- use the misaligned MC to redo the analysis and use the variation as an estimation of systematics.

See also a presentation from Sara at the TRK DPG that may be useful.

https://indico.cern.ch/event/787662/contributions/3361558/attachments/1815095/2966246/tkalignment_in_bph.pdf

