## This is a data flow diagram for how the data loads in the comment section on the politician details page.

Here's the sequence in order:

Page mounts with a politicianWeVoteId. The PoliticianRetrieveController fires an API call to get the politician's data, which includes candidate_list (all their candidacy records). When that response arrives, CandidateStore populates candidateListsByPoliticianWeVoteId[politicianWeVoteId].

CandidateStore emits a change because it just received new data. PoliticianPositionRetrieveController is listening — its onCandidateStoreChange fires, sees that a candidate list now exists (line 54), and calls positionsFirstRetrieve().

positionsFirstRetrieve iterates through the candidate records and fires CandidateActions.positionListForBallotItemPublic(candidateWeVoteId) for each one — these are the actual API calls to the server asking "give me all public endorsements for this candidate."

That same CandidateStore emit from step 2 also triggers onCandidateStoreChange in PoliticianDetailsPage. The page calls getAllCachedPositionsByPoliticianWeVoteId — but positions haven't been fetched yet (step 3 just initiated the calls). So the function walks the candidate list, finds nothing in allCachedPositionsAboutCandidates for each candidate, and returns [].

API responses arrive (from the calls made in step 3). CandidateStore's reducer handles positionListForBallotItem, populates allCachedPositionsAboutCandidates[candidateWeVoteId] with the actual position objects, and emits again.

onCandidateStoreChange fires again in PoliticianDetailsPage. Now getAllCachedPositionsByPoliticianWeVoteId finds real data in the dictionaries and returns [...actual positions...].

``` mermaid
sequenceDiagram
    participant Page as PoliticianDetailsPage
    participant PRC as PoliticianRetrieveController
    participant PPRC as PoliticianPositionRetrieveController
    participant API as API Server
    participant CS as CandidateStore

    Note over Page: Step 1: Page mounts with politicianWeVoteId

    PRC->>API: politicianRetrieve(politicianWeVoteId)
    API-->>CS: Response: politician data + candidate_list
    CS->>CS: Populate candidateListsByPoliticianWeVoteId[politicianWeVoteId]
    CS->>CS: Emit change

    Note over CS: Step 2: CandidateStore emits — two listeners react

    par Listener A: PoliticianPositionRetrieveController
        CS-->>PPRC: onCandidateStoreChange()
        PPRC->>PPRC: Sees candidate list exists
        PPRC->>PPRC: Calls positionsFirstRetrieve()
        Note over PPRC: Step 3: Fire position API calls
        loop For each candidate with future/recent election
            PPRC->>API: positionListForBallotItemPublic(candidateWeVoteId)
        end
    and Listener B: PoliticianDetailsPage
        CS-->>Page: onCandidateStoreChange()
        Note over Page: Step 4: Page reads store
        Page->>CS: getAllCachedPositionsByPoliticianWeVoteId()
        CS-->>Page: Returns [] (positions not fetched yet)
        Page->>Page: setState({ allCachedPositionsForThisPolitician: [] })
    end

    Note over API: Step 5: API responses arrive

    API-->>CS: positionListForBallotItem response (with positions)
    CS->>CS: Populate allCachedPositionsAboutCandidates[candidateWeVoteId]
    CS->>CS: Emit change

    Note over Page: Step 6: Page re-reads store

    CS-->>Page: onCandidateStoreChange()
    Page->>CS: getAllCachedPositionsByPoliticianWeVoteId()
    CS-->>Page: Returns [...actual positions...]
    Page->>Page: setState & render endorsements
  ```
