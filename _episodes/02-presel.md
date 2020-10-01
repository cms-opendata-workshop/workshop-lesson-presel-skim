---
title: "Preselection in OpenData NanoAOD"
teaching: 10
exercises: 0
questions:
- "What selections have been made when producing NanoAOD files?"
objectives:
- "Summarize selections on objects made during NanoAOD production"
keypoints:
- "Only objects deemed to be *interesting* survive to NanoAOD files"
---

Let's review the selections made in the production of the NanoAOD files for this exercise.
Data coming off the detector is already skimmed by the hardware triggers, and some further
skimming is done when creating object collections, but we will concentrate on the final step
of producing an "end user" ROOT file.

## Triggers

The trigger list is slimmed to only 3 paths for these NanoAOD samples! In order to perform
other analyses that required other triggers, new NanoAOD would need to be produced. This is
an example of "analysis-specific" code verses more general CMS NanoAOD that contains the
pass information for many more trigger paths. In the trigger manipulation lesson you learned
methods to find trigger paths that could be added to the "interestingTriggers" list below
for other physics analyses.

~~~
const static std::vector<std::string> interestingTriggers = {
    "HLT_IsoMu24_eta2p1",
    "HLT_IsoMu24",
    "HLT_IsoMu17_eta2p1_LooseIsoPFTau20",
};

// Trigger results
Handle<TriggerResults> trigger;
iEvent.getByLabel(InputTag("TriggerResults", "", "HLT"), trigger);
auto psetRegistry = edm::pset::Registry::instance();
auto triggerParams = psetRegistry->getMapped(trigger->parameterSetID());
TriggerNames triggerNames(*triggerParams);
TriggerResultsByName triggerByName(&(*trigger), &triggerNames);

for (size_t i = 0; i < interestingTriggers.size(); i++) {
  value_trig[i] = false;
}
const auto names = triggerByName.triggerNames();
for (size_t i = 0; i < names.size(); i++) {
  const auto name = names[i];
  for (size_t j = 0; j < interestingTriggers.size(); j++) {
    const auto interest = interestingTriggers[j];
    if (name.find(interest) == 0) {
      const auto substr = name.substr(interest.length(), 2);
      if (substr.compare("_v") == 0) {
        const auto status = triggerByName.state(name);
        if (status == 1) {
          value_trig[j] = true;
          break;
        }
      }
    }
  }
}
~~~
{: .language-cpp}

## Momentum thresholds

The objects have a variety of thresholds that reflect the simplicity of reconstructing each type.

 * Muons: pT > 3 GeV
 * Electrons & photons: pT > 5 GeV
 * Taus: pT > 15 GeV
 * Jets: pT > 15 GeV

Objects with momenta smaller than these thresholds might suffer from more uncertain reconstruction, and
"fakerates" (misidentification of spurious signals as objects) grow at very low momentum. In fact,
some object identification algorithms are intended for even higher momentum thresholds -- the electron
and photon identification algorithms used here are [recommended for pT > 15 GeV](https://twiki.cern.ch/twiki/bin/view/CMSPublic/EgammaPublicData), Given the larger uncertainties in the jet energy corrections
for low momentum jets, analysts are recommended to select jets of pT > 20 GeV or 30 GeV for the best
performance.

## Generated particles

In `AOD2NanoAOD.cc`, only leptons and photons that are matched to one of the reconstructed leptons or
photons are stored. This is a very stringent "preselection" on the list of generated particles. Many
analyses benefit from storing more of the generated particles for studies or decay mode selection.

~~~
if (!isData) {
  Handle<GenParticleCollection> gens;
  iEvent.getByLabel(InputTag("genParticles"), gens);

  value_gen_n = 0;
  std::vector<GenParticle> interestingGenParticles;
  for (auto it = gens->begin(); it != gens->end(); it++) {
    const auto status = it->status();
    const auto pdgId = std::abs(it->pdgId());
    if (status == 1 && pdgId == 13) { // muon
      interestingGenParticles.emplace_back(*it);
    }
    if (status == 1 && pdgId == 11) { // electron
      interestingGenParticles.emplace_back(*it);
    }
    if (status == 1 && pdgId == 22) { // photon
      interestingGenParticles.emplace_back(*it);
    }
    if (status == 2 && pdgId == 15) { // tau
      interestingGenParticles.emplace_back(*it);
    }
  }

  // Match muons with gen particles and jets
  for (auto p = selectedMuons.begin(); p != selectedMuons.end(); p++) {
    // Gen particle matching
    auto p4 = p->p4();
    auto idx = findBestVisibleMatch(interestingGenParticles, p4);
    if (idx != -1) {
      auto g = interestingGenParticles.begin() + idx;
      value_gen_pt[value_gen_n] = g->pt();
      value_gen_eta[value_gen_n] = g->eta();
      value_gen_phi[value_gen_n] = g->phi();
      value_gen_mass[value_gen_n] = g->mass();
      value_gen_pdgid[value_gen_n] = g->pdgId();
      value_gen_status[value_gen_n] = g->status();
      value_mu_genpartidx[p - selectedMuons.begin()] = value_gen_n;
      value_gen_n++;
    }

    // Jet matching
    value_mu_jetidx[p - selectedMuons.begin()] = findBestMatch(selectedJets, p4);
  }
  //...similar for other particles...
}
~~~
{: .language-cpp}

All of these selections on objects are intended to produce a compact, manageable-size dataset for the
Higgs->Tau Tau analysis that we will study more deeply in the next lesson. Understanding the selection
criteria applied allows the user to make changes to produce files for other analyses. 

{% include links.md %}

