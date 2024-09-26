---
name: Bite Ballot
tools: [Python, Flask, PostgreSQL, SQLAlchemy, HTMx, hyperscript, HTML, JavaScript, jinja]
image: https://cdn-b.saashub.com/images/app/screenshots/246/ieaz40q4t2zk/other-original.PNG
description: Easily elect the best restaurant for your group.
# external_url: https://www.biteballot.app
---

# Bite Ballot

![ballot]({{ site.baseurl }}/assets/image/ballot.png)

A seemingly simple web app for voting on where to eat. But you'll find a rich set of features exactly when you need them.

- Need to invite someone to your ballot over a messaging app? <strong>Send them the invite link.</strong>
- Want to invite someone in person? <strong>Have them scan the QR code.</strong>
- Need to see what ballot options others are adding? <strong>Ballots update in real-time.</strong>
- Someone forgot to submit their votes? <strong>Move to end voting, and have someone second it.</strong>
- Yada yada yada.

![qrcode]({{ site.baseurl }}/assets/image/qrcode.png)

## UX for joining a ballot

This is going to get complicated. So bear with me, or skip ahead to the code.

There are many possible situations that can happen when a person visits a join link, and they each need to be handled differently.
This creates complexity in the logic of the app in exchange for a smooth user experience.
Upon visiting a join link, the landing page is always shown along with various CTAs depending on the situation:

First, if the ballot they're trying to join is abandoned or already finished, they are informed, then
- ... if they don't have a user, they are asked to create a new ballot.
- ... if they do have a user, they are presented with a button to return to their current ballot.

![joining_not_exists]({{ site.baseurl }}/assets/image/joining_not_exists.png)

If, on the other hand, they're trying to join an active ballot, these cases are possible:
- If they don't have a user, they are allowed to join after giving their name.
- If they do have a user, and
  - ... they're currently in a completed ballot, they will be presented with the option to either join the ballot or view their previous ballot results.
  - ... they're currently in a different ballot, they will be asked if they want to abandon that ballot.
  - ... they're already in the ballot they're trying to join, they will be informed and presented with a button to return to the ballot.

![joining_previous_results]({{ site.baseurl }}/assets/image/joining_previous_results.png)

Phew, that was a lot! Despite the complexity, I was able to keep the code relatively simple, utilizing just a couple unnested if-statements.
Here's the entire `/join-ballot` handler:

```python
@appb.route('/join-ballot/<ballot_id>')
def join_ballot(ballot_id):
    ballot_uuid = uuid.UUID(ballot_id)
    ballot_join_url = ballot_service.get_join_url(ballot_id)
    device = get_device()

    if not ballot_service.is_ballot_active(ballot_uuid) or not device:
        can_join = ballot_service.is_ballot_active(ballot_uuid)
        return render_template('index.html.jinja',
                               in_join_flow=True,
                               ballot_join_url=ballot_join_url if can_join else None,
                               alert_cant_join=not can_join,
                               no_user=not device)

    user: User = device.user

    join_confirmed = request.args.get('join_confirmed')
    if join_confirmed:
        ballot_service.join_ballot(ballot_uuid, user)
        response = Response()
        response.headers['HX-Redirect'] = url_for('appb.index')
        return response

    return render_template('index.html.jinja',
                           in_join_flow=True,
                           ballot_join_url=ballot_join_url,
                           already_in=user.current_ballot.id == ballot_uuid,
                           warn_join=ballot_service.is_ballot_active(user.current_ballot.id),
                           in_ballot_result=user.current_ballot and user.current_ballot.result)
```

At almost 30 lines, it's starting to get long, but for now it remains readable enough.

## The voting algorithm

![voting]({{ site.baseurl }}/assets/image/screenshot_voting.png)

I extended the Approval Voting system, which allows voters to approve ("OK") multiple candidates -- the winner being the candidate with the most OKs.
For Bite Ballot, vetoes and prefers are also available.
Then, when voting is complete, for each candidate that has at least one OK or prefer vote:

1. Filter the candidates with the least number of vetoes.
2. If there are ties, filter the ones with the most number of prefers.
3. If there are still ties, filter the ones with the most number of OKs.
4. If there are still ties, pick one at random.

This ensures one restaurant always wins and that everyone's preferences are accounted for fairly.

The code I write reads as close to its description as possible:

```python
def _calculate_finalists(candidate_vote_counts: dict, ballot_result: BallotResult) -> list[Restaurant]:
    """
    Filters the least-vetoed options. Then filters the most-preferred options.
    Then filters the most-okayed options. What's left are the finalists.
    :return: A list of the finalists.
    """
    least_vetoed_restaurants = _filter_restaurants(candidate_vote_counts, min, VoteType.VETO)
    ballot_result.least_vetoed = [restaurant.name for restaurant in least_vetoed_restaurants.keys()]

    most_preferred_restaurants = _filter_restaurants(least_vetoed_restaurants, max, VoteType.PREFERRED)
    ballot_result.most_preferred = [restaurant.name for restaurant in most_preferred_restaurants.keys()]

    most_okayed_restaurants = _filter_restaurants(most_preferred_restaurants, max, VoteType.OKAY)
    ballot_result.most_okayed = [restaurant.name for restaurant in most_okayed_restaurants.keys()]

    return list(most_okayed_restaurants.keys())
```

And here's how the results are displayed to the user:

![ballot_results]({{ site.baseurl }}/assets/image/ballot_results.png)

## Software design

Bite Ballot was designed to operate as a single-page app (SPA). To ensure rapid development and a minimal codebase for the required functionality, lightweight frameworks and libraries were chosen.
Flask was chosen over Django. HTMx + hyperscript was chosen over React.

#### HATEOAS

The combination of HTMx and jinja was used to achieve Hypermedia As The Engine Of Application State (HATEOAS) and true Representational State Transfer (REST).
In other words, responses from the backend would contain some new HTML -- the new state of part of the application -- that resulted from handling the request.
This new HTML, which contained controls that could send further requests, would then be swapped into the Document Object Model (DOM). And back and forth it goes.

#### Long polling

Long polling was used as a simple method for the frontend to receive real-time updates from the backend without having to use web sockets.
This is distinct from traditional polling where an HTTP request is made at regular intervals and the backend responds immediately.
In long polling, the HTTP connection to the server is kept open for long periods.
Then, when another voter updates your ballot, say by adding a ballot option, the backend is able to immediately respond to your long-poll request with the updated ballot.

If, on the other hand, no changes are made within a certain timeout, the backend just responds with a 204 (no content), and the frontend starts another long poll request.

## Future

- User accounts
  - Saved restaurants
  - In-app ballot invites
- Restaurant auto-complete/suggestions
- Links to yelp
- Map
- Yelp integration
  - Stars
  - Food type
  - Hours
  - Menu
- Mobile apps
- Slack app

## Credits

Market research, design, engineering, QA, marketing: Me

Special thanks to my early testers: Nhi Tu, Timothy Villanueva, Ajani Spicer-grose, Yichen Wang, Brendan Wolf, PJ Quesada, Alexandra Chua, Josh Beck

<p class="text-center">
{% include elements/button.html link="https://www.biteballot.app" text="Bite Ballot" %}
</p>