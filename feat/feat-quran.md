```
A feature in the app where the user can listen/recite the holy quran.
Every verse is stored locally and every translation of the verse is stored locally as well.
```

Feature list inside quran:
1. ability to read quran in 2 view modes. 
   a. translation view: This is where people read the ayah and translation per verse in the language they chose. similar to this image. ![[Pasted image 20260708114725.png]]
   b. reading view: This is where users do not see any translations, just ayahs in a whole, similar to this image. ![[Pasted image 20260708114609.png]]
2. ability to listen to quran in 2 modes. 
   a. verse by verse mode via quran cloud API.
   b. full surah mode via API as well.
   - highlighting ayah being played in both modes.
   - when the user clicks an ayah, they have a popup asking to select from play this ayah or play from this ayah to the end of surah.
   - when the user chose full surah mode, there should be a media player similar to spotify with a progress bar and the reciter and the surah name. basically everything that spotify progress bar (or whatever that is called) has with full functionality (pause/resume/etc.).
   - ability to choose from list of reciters.
   - when a user chooses a different reciter while a surah is being played and then pauses and plays the surah again, the surah starts from beginning in the newly selected reciter.