# Tutorial: Building a Simple Tour

In this tutorial, we'll create a brief tour that shows a user how the site preview works.

It's not a comprehensive tutorial, but walks you through many of the different tasks that you'll encounter when developing a tour using Guided Tours.

For more complete technical information, please see the existing tours (linked to [in the README.md](../README.md)), the [API documentation](./API.md), and the [architecture documentation](./ARCHITECTURE.md).

**PLEASE NOTE:** This tour is merely an example. It is not a real tour that we're showing to users. The step sequence and copy especially didn't go through multiple iterations like we'd normally do with a new tour.

## What Do We Want To Achieve?

Before we think about what steps we want our tour to have, we should take a step back and think about **what goals we want to achieve** with our tour.

In this case, we might want to make sure new users understand where the site preview is and how it works.

In addition, we should think about how to **measure reaching this goal**. Here, we would hope that with our tour active, we would soon see an increase the usage of the site preview by new users.

Note: we've found it helpful for a developer and a designer (and possibly other roles, depending on your goals) to team up.

## Decide for Tour Steps

With that in mind, we can come up with this first draft for a set of steps:

1. Ask the user whether they want to learn how to preview their site.
2. Point to the site preview and tell the user to click their site's title so that the preview opens.
3. When the preview is showing, explain that this is the site preview.
4. Show the user how to exit the preview and wait for them to do it.
5. Finish the tour.

We could have included a step — or even multiple steps — to explain the different device sizes or the SEO upgrade. But it's advisable to start off with tours that are as short as possible to see how they perform. We can always come back and add additional steps.

And in that spirit, we can iterate over the steps once more to reduce them. We don't **really** need steps to ask whether a user wants to take the tour or not, and we don't **really** need a finishing step. Also, we could exit the tour while the preview is still open so the user can explore it without a tour step showing.

With these thoughts, we can come up with these steps:

1. Point to the site preview and tell the user to click their site's title to learn about the site preview. Provide a "Quit" button though so it's obvious what to do if they don't want to take the tour.
2. Show the user how to exit the preview, but finish the tour with the preview staying open.

And with that, we can now think about what will trigger our tour.

## Decide for Triggers

To start the tour, we need to provide a list of paths and a trigger function.

We want our tour to start when the user is looking at their stats. We therefore use `[ '/stats' ]` as the path. Note that in the tour implementation, we can provide a single path as just a string as opposed to an array.

We only want our tour to show for users who have registered in the past 7 days. We express that by giving the `Tour` element an appropriate `when` attribute.

## Write the Tour

Now we're ready to actually start writing our tour.

### Scaffolding, etc.

First we'll need to create a file for our tour, then add our essential boilerplate, which comprises the imports and the `makeTour` wrapping. For instance:

```JavaScript
/**
 * External dependencies
 */
import React from 'react';
import { translate } from 'i18n-calypso';
import {
	overEvery as and,
} from 'lodash';

/**
 * Internal dependencies
 */
import Gridicon from 'components/gridicon';
import {
	makeTour,
	Tour,
	Step,
	ButtonRow,
	Quit,
	Continue,
} from 'layout/guided-tours/config-elements';
import {
	isNewUser,
	isEnabled,
	isSelectedSitePreviewable,
} from 'state/ui/guided-tours/contexts';
import { isPreviewShowing } from 'state/ui/selectors';

export const TutorialSitePreviewTour = makeTour(
);
```

Now add that tour to the [tour list](../config.js):

```diff
 …
 import { SiteTitleTour } from 'layout/guided-tours/tours/site-title-tour';
+import { TutorialSitePreviewTour } from 'layout/guided-tours/tours/tutorial-site-preview-tour';

 export default combineTours( {
     …
     siteTitle: SiteTitleTour,
+    tutorialSitePreview: TutorialSitePreviewTour,
```

And add a [feature flag](https://github.com/Automattic/wp-calypso/blob/1cdd7380bb457dd4653fad1f27a6a53065d64c24/config/development.json#L46-L49) for the appropriate environment(s), such as `"guided-tours/tutorial-site-preview": true,`.

### The Tour element

Now in between the `makeTour` parantheses, create the tour element:

```JSX
export const TutorialSitePreviewTour = makeTour(
<Tour
	name="sitePreview"
	version="20170104"
	path="/stats"
	when={ and(
		isEnabled( 'guided-tours/main' ),
		isNewUser,
		) }
	>
	<!-- this is where the tour steps go ... -->
</Tour>
);
```

- `name` is the tour's name. It must match the key used in [config.js](../config.js), as we use it to refer to the tour inside the Guided Tours system (`combineTours`) and to force the tour to start via the query arg (`?tour=<TOURNAME>`).
- `version` is the tour's version. The value is yours to choose, though we usually use the date we created the tour in YYYYMMDD format.
- `path` is the path part of the URL that we want to trigger the tour on. Can also be an array.
- `when` is a boolean function that the framework tests to see whether the tour should be triggered or not. The first check should be checking whether the feature flag for this tour is enabled.

### Add an A/B Test

To assess the impact of a tour, it can be helpful to run it as an A/B test. If the user is in the test group we trigger the tour. If they're in the control group we don't trigger the tour. After you've collected some data, you'll hopefully be able to gauge the impact that the tour has on the metric(s) you're interested in.

Open up `client/lib/abtest/active-tests.js` and add a new test such as this one:

```JavaScript
	designShowcaseWelcomeTour: {
		datestamp: '20170101',
		variations: {
			enabled: 0,
			disabled: 100,
		},
		defaultVariation: 'disabled',
		allowExistingUsers: true,
	},
```

Note that we've set the `enabled` variation to 0% so we don't show the tour to any user until we've tested it thoroughly.

Now we need to make sure the tour only triggers if the user in the `enabled` variant.

First, add an import for `isAbTestInVariant` to the list of things we import from `state/ui/guided-tours/contexts`.

Now, use the import in the `Tour` element's `when` attribute like so:

```JavaScript
when={ and(
	isNewUser,
	isEnabled( 'guided-tours/main' ),
	isAbTestInVariant( 'tutorialSitePreviewTour', 'enabled' ),
	) }
```

Note that we want to put the call to `isAbTestInVariant` last — it puts users into an A/B test variant, and having later parts of the function return false would taint our results.

In a nutshell, we want to assign the user to an A/B test variant if and only if the tour would have triggered based on all the other conditions.

## Adding the First Step

Now let's insert the first `<Step>` element into the `<Tour>`. It looks like this:

```JSX
<Step name="init"
	target="site-card-preview"
	arrow="top-left"
	placement="below"
	when={ isSelectedSitePreviewable }
	scrollContainer=".sidebar__region"
>
	<p>
		{
			translate( "This shows your currently {{strong}}selected site{{/strong}}'s name and address.", {
				components: {
					strong: <strong />,
				}
			} )
		}
	</p>
	<Continue click step="close-preview" target="site-card-preview">
		{
			translate( "Click {{strong}}your site's name{{/strong}} to continue.", {
				components: {
					strong: <strong />,
				},
			} )
		}
	</Continue>
	<ButtonRow>
		<Quit subtle>{ translate( 'No, thanks.' ) }</Quit>
	</ButtonRow>
</Step>
```

A few notes:

- The first step of a tour needs to have a name of `init` to be recognizable as the first step by the framework.
- `isSelectedSitePreviewable` guards us against showing this step if the current site cannot be previewed.
- The `target` is the DOM element the step should be "glued" to or will point at. There are two ways to do that: either the element has a `data-tip-target` attribute, and we pass that name, or we pass a CSS selector that selects that element (cf. method `targetForSlug`). In this case, it's a `data-tip-target`.
- The `scrollContainer` tells the framework which container it should attempt to scroll in case the `target` isn't visible. In this case, the framework will attempt to scroll the sidebar until the site preview button is in view.
- `translate` calls: we'd add those only after multiple iterations over the copy. Once you merge something with a `translate` call into `master`, the strings will be translated -- and we don't want to waste anyone's time with translating strings that will still change a few times.
- The `Continue` steps attributes basically say: when the user `click`s the `target`, proceed to the step called `close-preview` (the next step, below).
- The `ButtonRow` with the `Quit` button doesn't really look nice, but it's important to provide a way for the user to get out of the tour. The framework will quit a tour if it believes that the user is trying to navigate away from it, but in this case we thought an explicit way to quit would be good to provide.

## Adding the Second Step

Now we add the second step after the first one:

```JSX
<Step name="close-preview"
	placement="center"
	when={ and( isSelectedSitePreviewable, isPreviewShowing ) }
>
	<p>
		{ translate( "Take a look around — and when you're done, close the site preview using the {{icon/}}. " +
			'You can come back here anytime.', {
				components: { icon: <Gridicon icon="cross" /> }
			}
		) }
	</p>
	<ButtonRow>
		<Quit primary>
			{ translate( 'Got it.' ) }
		</Quit>
	</ButtonRow>
</Step>
```

And a note:

- This time, we'll guard against `isSelectedSitePreviewable` and also `isPreviewShowing` -- this makes sure that the preview sheet is actually visible when we show this step.

## Finished!

You can see the finished tour in action by going to a URL like `http://calypso.localhost:3000/stats/insights/SITE_URL?tour=tutorialSitePreview`.

You'll have to replace SITE_URL with the URL of one of your sites.

## A Few More Tips

### Custom Styles

It's possible to pass custom styles to a step like so:

```JSX
<Step
	...
	style={ { marginTop: '-15px', zIndex: 1 } }
>
```

In this case, the automatic positioning was correctly pointing at the target, but the visual weight was off by a few pixels.

**Caveat:** Please use this sparingly or not at all. Minor fixes such as those presented above are fine, but the general Guided Tours style should be kept consistent across Calypso and all its tours.

### Adding a Delay

If your first step displays using an animation, you can effectively delay the start of the tour by adding an animationDelay to the `Step`'s styles. This could buy some important time for letting things render properly.

```JSX
style={ {
	animationDelay: '10s',
} }
```

**Caveat:** Please use this sparingly or not at all. Ask yourself whether what you are attempting to solve can't be solved with better selectors for `when`.

### Dealing with async

_Context, which you are free to skip:_ Roughly speaking, there are two kinds of on-demand loading in Calypso: code and data. **Code** is loaded in chunks thanks to code splitting, a feature powered by webpack; most commonly, chunks have to be loaded when the user requests to navigate to a new section, though it is possible to manually break down a section into different chunks and load them [asynchronously][async-load] for performance gains, as is [done in the Editor][async-load-usage]. **Data** is loaded, typically, by firing AJAX requests and handling their responses in or close to the _action_ layer of the Redux subsystem. Practically speaking, this means that a typical request for data will generate a series of Redux actions such as `REQUEST_FOOS`, `RECEIVE_FOOS`, `REQUEST_FOOS_SUCCESS`, `REQUEST_FOOS_FAILURE`. Specific parts of the UI will then "subscribe" to changes using `react-redux`'s `connect` helper.

_The issue:_ Guided Tours has built-in support for section-wise code splitting, meaning that it knows how to wait when Calypso has to wait for a chunk to arrive before rendering a section's UI, etc. However, upon a user's request to navigate to a new section, it is common that _both_ code and data need to be loaded before the UI is rendered. This duality can become confusing to developers and lead to undiagnosed bugs in new tours. Example:

1. A tour, `siteTitle`, has a step prompting the user to click _Settings_ on the sidebar.
2. Once the user navigates to `/settings` and that section is loaded, Guided Tours displays the next step, which points at a specific input field of that new view, prompting the user for the next action.

If neither the code chunk nor the site data required for `/settings` are available locally, there will be a race condition whereby the new chunk will be loaded and run, thus clearing the `isSectionLoaded` flag in Guided Tours, thus confirming the transition from the previous step to the new one and rendering that step's contents, **but** because the data itself hasn't been received yet, the full UI for `/settings` doesn't exist yet, so the new step won't have any `target`s (or won't have the correct/final ones) to position itself next to. The result is a grossly mispositioned step that will manifest itself to the user either a broken UI or as though the Guided Tour has abruptly quit. Usually, this will go **unnoticed in development**, as chunks are loaded locally with no latency.

_The [fix][pr-10521]:_ We make Guided Tours "subscribe" to the corresponding data requests using its all-purpose [`actionLog`][action-log]: simply add the action type signaling the satisfaction of a data need — _e.g._, `RECEIVE_FOOS` or `REQUEST_FOOS_SUCCESS` — to the log's [white list][relevant-types]. Any change to `actionLog` triggers all of Guided Tours' view layer to update, thereby allowing a correct and timely positioning of steps.

[async-load]: https://github.com/Automattic/wp-calypso/blob/master/client/components/async-load/README.md
[async-load-usage]: https://github.com/Automattic/wp-calypso/blob/791003963e72c39589073b4de634bf946d1d288f/client/post-editor/editor-sidebar/index.jsx#L43
[pr-10521]: https://github.com/Automattic/wp-calypso/pull/10521
[action-log]: https://github.com/Automattic/wp-calypso/tree/791003963e72c39589073b4de634bf946d1d288f/client/state/ui/action-log
[relevant-types]: https://github.com/Automattic/wp-calypso/blob/791003963e72c39589073b4de634bf946d1d288f/client/state/ui/action-log/reducer.js#L18-L25
