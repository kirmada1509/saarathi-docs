## 1. Boot-Up Sequence (Data Caching)                                                                                                                                                      
                                                                                                                                                                                             
  To avoid querying SQLite for every single recommendation, the backend loads the entire dataset into memory once at startup.                                                                
                                                                                                                                                                                             
    sequenceDiagram                                                                                                                                                                          
        participant Nest as NestJS (app.module.ts)                                                                                                                                           
        participant Prisma as Prisma Service                                                                                                                                                 
        participant Store as In-Memory Cache (core/data.ts)                                                                                                                                  
                                                                                                                                                                                             
        Nest->>Nest: OnModuleInit()                                                                                                                                                          
        Nest->>Prisma: Fetch all Users & Flights                                                                                                                                             
        Prisma-->>Nest: SQLite records                                                                                                                                                       
        Nest->>Store: initializeStoreFromDb(prisma)                                                                                                                                          
        Note over Store: Parses & indexes flights by origin,<br/>destination, and user IDs into Maps.                                                                                        
        Store-->>Nest: Cache initialized (globalThis.__saarathi_store__)                                                                                                                     
                                                                                                                                                                                             
  • Entrypoint:  src/app.module.ts                                                                                                                                                           
  • Trigger:  OnModuleInit  hook.                                                                                                                                                            
  • Function called:  initializeStoreFromDb(this.prisma)  in data.ts.                                                                                                                        
      • Queries database using  prisma.user.findMany()  and  prisma.flight.findMany() .                                                                                                      
      • Coerces columns using  coerceUserRow()  and  coerceFlightRow()  to normalize strings/numbers.                                                                                        
      • Populates Maps of users, airports, flights indexed by origin ( flightsByOrigin ), and flights indexed by route ( flightsByRoute  i.e.  origin-destination ).                         
      • Saves this store globally inside  globalThis.__saarathi_store__ .                                                                                                                    
      • Subsequent requests access this data via  getStore() .                                                                                                                               
                                                                                                                                                                                             
  ──────                                                                                                                                                                                     
  ## 2. API Endpoints                                                                                                                                                                        
                                                                                                                                                                                             
  The backend exposes two HTTP REST endpoints:                                                                                                                                               
                                                                                                                                                                                             
  ### A.  GET /api/users                                                                                                                                                                     
                                                                                                                                                                                             
  • File: users.controller.ts                                                                                                                                                                
  • Flow: Directly queries the SQLite database using  prisma.user.findMany(...)  to list users. It bypasses the in-memory cache (a slight architectural inconsistency).                      
                                                                                                                                                                                             
  ### B.  POST /api/recommend  (Core Flow)                                                                                                                                                   
                                                                                                                                                                                             
  • Files: recommend.controller.ts, services/recommend.service.ts, services/recommend-single-leg.service.ts, services/recommend-multi-city.service.ts                                                                   
  • Flow: The controller validates the request payload and delegates the execution to the `RecommendService`. The service performs preference inference, applies perturbations, and then delegates the work to either `RecommendSingleLegService` or `RecommendMultiCityService` depending on the route type.                                                                                                                                                           
  ──────                                                                                                                                                                                     
  ## 3. Function Call Stack for  POST /api/recommend                                                                                                                                         
                                                                                                                                                                                             
  ### Step 1: Body Validation (Zod)                                                                                                                                                          
                                                                                                                                                                                             
  The incoming payload is validated against  RecommendRequestSchema :                                                                                                                        
                                                                                                                                                                                             
  •  userId : String (Required)                                                                                                                                                              
  •  requestText : String (Required)                                                                                                                                                         
  •  origin  /  destination : String (Optional)                                                                                                                                              
  •  cities : Array of Strings (Optional, triggers multi-city mode)                                                                                                                          
  •  perturbations : Array of active counterfactual chips (Optional)                                                                                                                         
  ──────                                                                                                                  ### Step 2: Preference Inference ( inferPreferences )                                                                                                                                      
                                                                                                                                                                                              
  • Function called:  inferPreferences(user, requestText)  inside preferences.ts.                                                                                                            
  • Behavior:                                                                                                                                                                                
      1. Retrieves base preference coefficients from structured fields (e.g. mapping  direct_preference :  strong -> 0.9 ,  moderate -> 0.55 ,  none -> 0.15 ).                              
      2. Splits  user.raw_history  on  |  into distinct phrases, and parses  requestText  into distinct sentences/clauses as additional input phrases.                                       
      3. Checks regex patterns (e.g.,  DIRECT_BOOST ,  COST_BOOST ,  REDEYE_AVOID ) against each phrase. Phrases matching from  requestText  are labeled with source  trip_description .         
      4. Embedding Fallback: If a phrase matches no regex, it loads  @xenova/transformers  ( all-MiniLM-L6-v2 ) in-process and runs  findEmbeddingMatch(phrase)  comparing its 384-          
      dimensional dot product similarity against pre-defined archetype phrases (threshold >0.78).                                                                                            
      5. Accumulates  directHits ,  costHits , and  convHits  to adjust direct/cost/convenience weights upwards by  +0.1  or  +0.15  per hit (capped at  1.0 ).                              
      6. Returns  InferredPreference  object containing weights and a descriptive  evidence  trail.                                                                                                                                                                    
                                                                                                                                                                                             
  ──────                                                                                                                                                                                     
  ### Step 3: Perturbation Application ( applyPerturbations )                                                                                                                                
                                                                                                                                                                                             
  • Function called:  applyPerturbations(basePref, perturbations)  inside counterfactuals.ts.                                                                                                
  • Behavior: Returns a cloned copy of preferences modified by client-side selections:                                                                                                       
      •  accept_one_stop  → lowers  direct_weight  to  0.15 , raises  max_layover_minutes  to  420 .                                                                                         
      •  bags_matter  → flags  bags_matter = true .                                                                                                                                          
      •  evening_ok  → sets  avoid_redeye = false .                                                                                                                                          
      •  ignore_loyalty  → clears preferred airlines array.                                                                                                                                  
                                                                                                                                                                                             
  ──────                                                                                                                                                                                     
  ### Step 4: Routing Execution (Single-Leg vs. Multi-City)                                                                                                                                  
                                                                                                                                                                                             
  #### Mode A: Single-Leg Route                                                                                                                                                              
                                                                                                                                                                                             
  1. Destination Parsing: Infers destination or cities list from  requestText  if not supplied explicitly (scans for 3-letter uppercase airport codes or matches city names to classify the route as single-leg or multi-city).                               
  2. Filtering & Scoring: Calls  filterAndRank(routeFlights, perturbedPref, opts)  inside ranking.ts.                                                                                        
      • Hard Constraints Funnel: Filters candidate flights sequentially:                                                                                                                     
          1. Origin match                                                                                                                                                                    
          2. Destination match                                                                                                                                                               
          3. Layover duration ≤  max_layover_minutes  (nonstop passes automatically)                                                                                                         
          4. Redeye filter: if  avoid_redeye  is true, removes departures between 10:00 PM and 5:00 AM UTC.                                                                                  
          5. Date flexibility range (if target date requested).                                                                                                                              
      • Utility Scoring Formula:                                                                                                                                                             
                                                                                                                                                                                             
                                                                                                                                                                                             
    Score = ⎛W    ·S      + W      ·S       + ⎛1 - W    ⎞·0.5·S     + W           ·0.3·S      + 0.2·S        + W       ·S       ⎞·DemandAdj·HolidayAdj + DayBonus                            
            ⎝ cost  price    direct  direct   ⎝     cost⎠      time    convenience      cabin        airline    baggage  baggage⎠                                                            
                                                                                                                                                                                             
     * *DayBonus*: Soft nudge of +0.15 if departure day matches a preferred day mentioned in `requestText` (e.g., "fly on a Friday").                                                        
                                                                                                                                                                                             
  • Dynamic Constraint Relaxation: If  ranked.length === 0 , the engine looks at  filterTrace.steps  to find the binding constraint (the step that removed the most flights). It             
  automatically relaxes layovers by 1.5x or removes redeye avoidance, runs  filterAndRank  again, and flags the recommendation as  [Relaxed] .                                               
                                                                                                                                                                                             
  3. Alternative Options Selection: Calls  selectAlternatives(ranked, user)  to select exactly five edge-case comparison flights (Cheapest, Fastest, Comfort upgrade, Flexible, and Shift    
  dates).                                                                                                                                                                                    
  4. Closed-Form Counterfactuals: Calls  computeCounterfactuals(ranked, basePref, routeFlights, opts)  inside counterfactuals.ts.                                                            
      • Price Thresholds (Closed-Form): For the top 3 challengers, it calculates the exact fare drop required to overtake the champion. Since the scoring formula is linear in price, it     
      solves:                                                                                                                                                                                
                                                                                                                                                                                             
                                                                                                                                                                                             
                ⎛Score         - Score          ⎞·priceRange                                                                                                                                 
                ⎝     champion        challenger⎠                                                                                                                                            
    priceDrop = ────────────────────────────────────────────                                                                                                                                 
                         W    ·DemandAdj·HolidayAdj                                                                                                                                          
                          cost                                                                                                                                                               
                                                                                                                                                                                             
     If the drop >60% of the challenger's price or requires a negative price, it flags it as unrealistic (`flips: false`).                                                                   
                                                                                                                                                                                             
  • Toggle Flips (Re-ranking): For other toggles, it re-runs  filterAndRank  with the perturbed preferences to verify if the winner changes.                                                 
                                                                                                                                                                                             
  #### Mode B: Multi-City Route                                                                                                                                                              
                                                                                                                                                                                             
  1. Permutations generation: Calls  optimizeRoute(cities, perturbedPref, stayDurations)  in multicity.ts.                                                                                   
      • Computes all possible city ordering permutations (e.g., A → B → C → A vs A → C → B → A).                                                                                             
  2. Optimal Pathing (Branch-and-Bound):                                                                                                                                                     
      • For each permutation, runs a branch-and-bound search.                                                                                                                                
      • Checks  satisfiesStayDuration(leg1, leg2, stayCity, stayDurations)  to ensure at least `stayDurations[city] * 24` hours are spent at each destination, falling back to 60 minutes if unspecified.
      • Prunes search branches if the maximum possible remaining score cannot exceed the best path found so far.                                                                             
  3. Synthetic Margin Formulation:                                                                                                                                                           
      • Creates a synthetic ranked array representing the score gap between the winner permutation and runner-up permutation. This is injected into the single-leg confidence calculator to  
      maintain consistent metrics.                                                                                                                                                           
                                                                                                                                                                                             
  ──────                                                                                                                                                                                     
  ### Step 5: Confidence Calculation ( computeConfidence )                                                                                                                                   
                                                                                                                                                                                             
  • Function called:  computeConfidence(ranked, perturbedPref)  inside confidence.ts.                                                                                                        
  • Behavior:                                                                                                                                                                                
      1. Computes the maximum possible theoretical score achievable under the current weights.                                                                                               
      2. Divides the champion's score by this maximum to output  matchPct  (0–100%).                                                                                                         
      3. Classifies strong vs. weak signals: a signal is strong only when both structured profile data and raw behavior (history/embeddings) agree on that preference dimension.             
      4. Rates confidence  tier  ( high ,  medium ,  low ) based on the score margin between the champion and runner-up:                                                                     
          • Margin >0.1 →  high                                                                                                                                                              
          • Margin <0.04 →  low                                                                                                                                                              
      5. Demotion Rules: Demotes the tier if there's a priority conflict (e.g., high sensitivity to both cost and comfort) or if there are zero strong signals.                              
      6. Floor Promotion: Promotes  low  to  medium  if  matchPct >= 80%  to avoid declaring a mathematically strong match as low confidence.                                                
                                                                                                                                                                                             
  ──────                                                                                                                                                                                     
  ### Step 6: Narrative Generation ( explain )                                                                                                                                               
                                                                                                                                                                                             
  • Function called:  explain(userId, requestText, pref, ranked, alternatives, counterfactuals, confidence)  inside explain.ts.                                                              
  • Behavior:                                                                                                                                                                                
      • Constructs a detailed prompt feeding in the user ID, original query, preferences evidence trail, top options, alternatives, counterfactuals, and confidence ratings.                 
      • Pipes the prompt to LangChain's  ChatGroq  wrapper targeting  llama-3.3-70b-versatile .                                                                                              
      • Fallback Plan: If  GROQ_API_KEY  is missing or the external API call fails, it immediately catches the error and generates a deterministic, formatted string using                   
      fallbackExplanation()  so the service never crashes.                                                                                                                                   
                                                                                                                                                                                             
  ──────                                                                                                                                                                                     
  ### Step 7: Response Construction                                                                                                                                                          
                                                                                                                                                                                             
  The selected recommendation service (`RecommendSingleLegService` or `RecommendMultiCityService`) collects the outputs, builds a structured execution trace (representing the stages shown in the frontend's debugging panel), and returns the response object. The controller then returns the JSON payload back to the Next.js frontend client:                                                                        
                                                                                                                                                                                             
    {                                                                                                                                                                                        
      "mode": "single-leg",                                                                                                                                                                  
      "verdict": { ...ScoredFlight },                                                                                                                                                        
      "ranked": [ ...ScoredFlight ],                                                                                                                                                         
      "preference": { ...InferredPreference },                                                                                                                                               
      "alternatives": [ ...Alternative ],                                                                                                                                                    
      "counterfactuals": [ ...Counterfactual ],                                                                                                                                              
      "confidence": { ...Confidence },                                                                                                                                                       
      "trace": [ ...TraceStage ],                                                                                                                                                            
      "explanation": "..."                                                                                                                                                                   
    }                                                                                                                                                                                        
    
