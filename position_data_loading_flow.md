## This is a data flow diagram for how the data loads in the comment section on the politician details page.

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
