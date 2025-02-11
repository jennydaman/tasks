<!--
@copyright Copyright (c) 2018 Team Popcorn <teampopcornberlin@gmail.com>
@copyright Copyright (c) 2019 Georg Ehrke <oc.list@georgehrke.com>
@copyright Copyright (c) 2019 Jakob Röhrl <jakob.roehrl@web.de>
@copyright Copyright (c) 2020 Raimund Schlüßler <raimund.schluessler@mailbox.org>

@author Team Popcorn <teampopcornberlin@gmail.com>
@author Georg Ehrke <oc.list@georgehrke.com>
@author Jakob Röhrl <jakob.roehrl@web.de>
@author Raimund Schlüßler <raimund.schluessler@mailbox.org>

@license GNU AGPL version 3 or any later version

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as
published by the Free Software Foundation, either version 3 of the
License, or (at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program. If not, see <http://www.gnu.org/licenses/>.

-->

<template>
	<div class="calendar-shares">
		<ul>
			<li class="app-navigation-entry__multiselect">
				<Multiselect
					id="users-groups-search"
					:options="usersOrGroups"
					:searchable="true"
					:internal-search="false"
					:max-height="600"
					:show-no-results="true"
					:placeholder="placeholder"
					:class="{ 'showContent': inputGiven, 'icon-loading': isLoading }"
					:user-select="true"
					open-direction="bottom"
					track-by="user"
					label="user"
					@search-change="findSharee"
					@change="shareCalendar" />
			</li>
			<!-- list of user or groups calendar is shared with -->
			<CalendarSharee v-for="sharee in calendar.shares"
				:key="sharee.uri"
				:sharee="sharee"
				:calendar="calendar" />
		</ul>
	</div>
</template>

<script>
import CalendarSharee from './CalendarSharee.vue'
import client from '../../services/cdav.js'

import Axios from '@nextcloud/axios'
import { generateOcsUrl } from '@nextcloud/router'
import Multiselect from '@nextcloud/vue/dist/Components/Multiselect'

import debounce from 'debounce'

export default {
	name: 'CalendarShare',
	components: {
		CalendarSharee,
		Multiselect,
	},
	props: {
		calendar: {
			type: Object,
			default() {
				return {}
			},
		},
	},
	data() {
		return {
			isLoading: false,
			inputGiven: false,
			usersOrGroups: [],
		}
	},
	computed: {
		placeholder() {
			return this.$t('tasks', 'Share with users or groups')
		},
		noResult() {
			return this.$t('tasks', 'No users or groups')
		},
	},
	mounted() {
		// This ensures that the multiselect input is in focus as soon as the user clicks share
		document.getElementById('users-groups-search').focus()
	},
	methods: {
		/**
		 * Share calendar
		 *
		 * @param {Object} data destructuring object
		 * @param {String} data.user the userId
		 * @param {String} data.displayName the displayName
		 * @param {String} data.uri the sharing principalScheme uri
		 * @param {Boolean} data.isGroup is this a group ?
		 * @param {Boolean} data.isCircle is this a circle?
		 */
		shareCalendar({ user, displayName, uri, isGroup, isCircle }) {
			const calendar = this.calendar
			uri = decodeURI(uri)
			user = decodeURI(user)
			this.$store.dispatch('shareCalendar', { calendar, user, displayName, uri, isGroup, isCircle })
		},

		/**
		 * Use the cdav client call to find matches to the query from the existing Users & Groups
		 *
		 * @param {String} query
		 */
		findSharee: debounce(async function(query) {
			const hiddenPrincipalSchemes = []
			const hiddenUrls = []
			this.calendar.shares.forEach((share) => {
				hiddenPrincipalSchemes.push(share.uri)
			})
			if (this.$store.getters.getCurrentUserPrincipal) {
				hiddenUrls.push(this.$store.getters.getCurrentUserPrincipal.url)
			}
			if (this.calendar.owner) {
				hiddenUrls.push(this.calendar.owner)
			}

			this.isLoading = true
			this.usersOrGroups = []

			if (query.length > 0) {
				const davPromise = this.findShareesFromDav(query, hiddenPrincipalSchemes, hiddenUrls)
				const ocsPromise = this.findShareesFromCircles(query, hiddenPrincipalSchemes, hiddenUrls)

				const [davResults, ocsResults] = await Promise.all([davPromise, ocsPromise])
				this.usersOrGroups = [
					...davResults,
					...ocsResults,
				]

				this.isLoading = false
				this.inputGiven = true
			} else {
				this.inputGiven = false
				this.isLoading = false
			}
		}, 500),
		/**
		 *
		 * @param {String} query The search query
		 * @param {String[]} hiddenPrincipals A list of principals to exclude from search results
		 * @param {String[]} hiddenUrls A list of urls to exclude from search results
		 * @returns {Promise<Object[]>}
		 */
		async findShareesFromDav(query, hiddenPrincipals, hiddenUrls) {
			let results
			try {
				results = await client.principalPropertySearchByDisplayname(query)
			} catch (error) {
				return []
			}

			return results.reduce((list, result) => {
				if (hiddenPrincipals.includes(decodeURI(result.principalScheme))) {
					return list
				}
				if (hiddenUrls.includes(result.url)) {
					return list
				}

				// Don't show resources and rooms
				if (!['GROUP', 'INDIVIDUAL'].includes(result.calendarUserType)) {
					return list
				}

				const isGroup = result.calendarUserType === 'GROUP'
				list.push({
					user: result[isGroup ? 'groupId' : 'userId'],
					displayName: result.displayname,
					icon: isGroup ? 'icon-group' : 'icon-user',
					uri: result.principalScheme,
					isGroup,
					isCircle: false,
					isNoUser: isGroup,
					search: query,
				})
				return list
			}, [])
		},
		/**
		 *
		 * @param {String} query The search query
		 * @param {String[]} hiddenPrincipals A list of principals to exclude from search results
		 * @param {String[]} hiddenUrls A list of urls to exclude from search results
		 * @returns {Promise<Object[]>}
		 */
		async findShareesFromCircles(query, hiddenPrincipals, hiddenUrls) {
			let results
			try {
				results = await Axios.get(generateOcsUrl('apps/files_sharing/api/v1') + 'sharees', {
					params: {
						format: 'json',
						search: query,
						perPage: 200,
						itemType: 'principals',
					},
				})
			} catch (error) {
				return []
			}

			if (results.data.ocs.meta.status === 'failure') {
				return []
			}
			let circles = []
			if (Array.isArray(results.data.ocs.data.circles)) {
				circles = circles.concat(results.data.ocs.data.circles)
			}
			if (Array.isArray(results.data.ocs.data.exact.circles)) {
				circles = circles.concat(results.data.ocs.data.exact.circles)
			}

			if (circles.length === 0) {
				return []
			}

			return circles.filter((circle) => {
				return !hiddenPrincipals.includes('principal:principals/circles/' + circle.value.shareWith)
			}).map(circle => ({
				user: circle.label,
				displayName: circle.label,
				icon: 'icon-circle',
				uri: 'principal:principals/circles/' + circle.value.shareWith,
				isGroup: false,
				isCircle: true,
				isNoUser: true,
				search: query,
			}))
		},
	},
}
</script>
