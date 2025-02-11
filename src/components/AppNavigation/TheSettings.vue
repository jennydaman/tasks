<!--
Nextcloud - Tasks

@author Raimund Schlüßler
@copyright 2018 Raimund Schlüßler <raimund.schluessler@mailbox.org>

This library is free software; you can redistribute it and/or
modify it under the terms of the GNU AFFERO GENERAL PUBLIC LICENSE
License as published by the Free Software Foundation; either
version 3 of the License, or any later version.

This library is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU AFFERO GENERAL PUBLIC LICENSE for more details.

You should have received a copy of the GNU Affero General Public
License along with this library.  If not, see <http://www.gnu.org/licenses/>.

-->

<template>
	<div class="reactive">
		<ul>
			<li>
				<label for="defaultCalendar">
					{{ $t('tasks', 'Default list') }}
				</label>
				<select id="defaultCalendar" v-model="defaultCalendarId">
					<option v-for="calendar in calendars"
						:key="calendar.id"
						:value="calendar.id">
						{{ calendar.displayName }}
					</option>
				</select>
			</li>
			<li class="headline">
				{{ $t('tasks', 'Visibility of Smart Collections') }}
			</li>
			<li v-for="collection in collections"
				:key="collection.id"
				class="collection">
				<div :class="collection.icon" class="icon" />
				<span class="label-container">
					<label :for="'visibilityCollection-' + collection.id" class="title">
						{{ collection.displayName }}
					</label>
				</span>
				<select :id="'visibilityCollection-' + collection.id"
					:value="collection.show"
					@change="setVisibility({ id: collection.id, show: +$event.target.value })">
					<option v-for="collectionOption in collectionOptions"
						:key="collectionOption.id"
						:value="collectionOption.id">
						{{ collectionOption.name }}
					</option>
				</select>
			</li>
		</ul>
	</div>
</template>

<script>
import moment from '@nextcloud/moment'

import { mapState, mapGetters, mapActions } from 'vuex'

export default {
	data() {
		return {
			collectionOptions: [
				{
					id: 0,
					name: this.$t('tasks', 'Hidden'),
				},
				{
					id: 1,
					name: this.$t('tasks', 'Visible'),
				},
				{
					id: 2,
					name: this.$t('tasks', 'Automatic'),
				},
			],
			dayOfMonth: moment().date(),
		}
	},
	computed: {
		defaultCalendarId: {
			get() {
				const cal = this.$store.getters.getDefaultCalendar
				return cal ? cal.id : ''
			},
			set(value) {
				this.$store.dispatch('setSetting', { type: 'defaultCalendarId', value })
			},
		},
		...mapState({
			collections: state => state.collections.collections,
		}),
		...mapGetters({
			calendars: 'getSortedWritableCalendars',
		}),
	},
	methods:
		mapActions([
			'setVisibility',
		]),
}
</script>
