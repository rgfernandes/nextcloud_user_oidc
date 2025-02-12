<!--
  - @copyright 2020 Christoph Wurst <christoph@winzerhof-wurst.at>
  -
  - @author 2020 Christoph Wurst <christoph@winzerhof-wurst.at>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program.  If not, see <http://www.gnu.org/licenses/>.
  -->

<template>
	<div>
		<div class="section">
			<h2>{{ t('user_oidc', 'OpenID Connect') }}</h2>
			<p>
				{{ t('user_oidc', 'Allows users to authenticate via OpenID Connect providers.') }}
			</p>
			<p>
				<input id="user-oidc-id4me"
					v-model="id4meState"
					type="checkbox"
					class="checkbox"
					:disabled="loadingId4Me"
					@change="onId4MeChange">
				<label for="user-oidc-id4me">{{ t('user_oidc', 'Enable ID4me') }}</label>
			</p>
		</div>
		<div class="section">
			<h2>
				{{ t('user_oidc', 'Registered Providers') }}
				<Actions>
					<ActionButton icon="icon-add" @click="showNewProvider=true">
						{{ t('user_oidc', 'Register new provider') }}
					</ActionButton>
				</Actions>
			</h2>

			<Modal v-if="showNewProvider" :can-close="false">
				<div class="providermodal__wrapper">
					<h3>{{ t('user_oids', 'Register a new provider') }}</h3>
					<p class="settings-hint">
						{{ t('user_oidc', 'Configure your provider to redirect back to {url}', {url: redirectUrl}) }}
					</p>
					<SettingsForm :provider="newProvider" @submit="onSubmit" @cancel="showNewProvider=false" />
				</div>
			</Modal>

			<div class="oidcproviders">
				<p v-if="providers.length === 0">
					{{ t('user_oidc', 'No providers registered.') }}
				</p>
				<div v-for="provider in providers"
					v-else
					:key="provider.id"
					class="oidcproviders__provider">
					<div class="oidcproviders__details">
						<b>{{ provider.identifier }}</b><br>
						{{ t('user_oidc', 'Client ID') }}: {{ provider.clientId }}<br>
						{{ t('user_oidc', 'Discovery endpoint') }}: {{ provider.discoveryEndpoint }}
					</div>
					<Actions>
						<ActionButton icon="icon-rename" @click="updateProvider(provider)">
							{{ t('user_oidc', 'Update') }}
						</ActionButton>
					</Actions>
					<Actions>
						<ActionButton icon="icon-delete" @click="onRemove(provider)">
							{{ t('user_oidc', 'Remove') }}
						</ActionButton>
					</Actions>
				</div>
			</div>

			<Modal v-if="editProvider" :can-close="false">
				<div class="providermodal__wrapper">
					<h3>{{ t('user_oidc', 'Update provider settings') }}</h3>
					<SettingsForm :provider="editProvider"
						:update="true"
						:submit-text="t('user_oidc', 'Update provider')"
						@submit="onUpdate"
						@cancel="editProvider=null" />
				</div>
			</Modal>
		</div>
	</div>
</template>

<script>
import axios from '@nextcloud/axios'
import { generateUrl } from '@nextcloud/router'
import { showError } from '@nextcloud/dialogs'
import Actions from '@nextcloud/vue/dist/Components/Actions'
import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import Modal from '@nextcloud/vue/dist/Components/Modal'

import logger from '../logger'
import SettingsForm from './SettingsForm'

export default {
	name: 'AdminSettings',
	components: {
		SettingsForm,
		Actions,
		ActionButton,
		Modal,
	},
	props: {
		initialId4MeState: {
			type: Boolean,
			required: true,
		},
		initialProviders: {
			type: Array,
			required: true,
		},
		redirectUrl: {
			type: String,
			required: true,
		},
	},
	data() {
		return {
			id4meState: this.initialId4MeState,
			loadingId4Me: false,
			providers: this.initialProviders,
			newProvider: {
				identifier: '',
				clientId: '',
				clientSecret: '',
				discoveryEndpoint: '',
				settings: {
					uniqueUid: true,
					checkBearer: false,
				},
			},
			showNewProvider: false,
			editProvider: null,
		}
	},
	methods: {
		async onId4MeChange() {
			logger.info('ID4me state changed', { enabled: this.id4meState })

			this.loadingId4Me = true
			try {
				const url = generateUrl('/apps/user_oidc/provider/id4me')

				await axios.post(url, {
					enabled: this.id4meState,
				}
				)
			} catch (error) {
				logger.error('Could not save ID4me state: ' + error.message, { error })
				showError(t('user_oidc', 'Could not save ID4me state: ' + error.message))
			} finally {
				this.loadingId4Me = false
			}
		},
		updateProvider(provider) {
			this.editProvider = { ...provider }
		},
		updateProviderCancel() {
			this.editProvider = null
		},
		async onUpdate(provider) {
			logger.info('Update oidc provider', { data: provider })

			const url = generateUrl(`/apps/user_oidc/provider/${provider.id}`)
			try {
				await axios.put(url, provider)
				this.editProvider = null
				const index = this.providers.findIndex((p) => p.id === provider.id)
				this.$set(this.providers, index, provider)
			} catch (error) {
				logger.error('Could not update the provider: ' + error.message, { error })
				showError(t('user_oidc', 'Could not update the provider:') + ' ' + (error.response?.data?.message ?? error.message))
			}
		},
		async onRemove(provider) {
			logger.info('Remove oidc provider', { provider })

			const url = generateUrl(`/apps/user_oidc/provider/${provider.id}`)
			try {
				await axios.delete(url)

				this.providers = this.providers.filter(p => p.id !== provider.id)
			} catch (error) {
				logger.error('Could not remove a provider: ' + error.message, { error })
				showError(t('user_oidc', 'Could not remove provider: ' + error.message))
			}
		},
		async onSubmit() {
			logger.info('Add new oidc provider', { data: this.newProvider })

			const url = generateUrl('/apps/user_oidc/provider')
			try {
				const response = await axios.post(url, this.newProvider)

				this.providers.push(response.data)

				this.newProvider.identifier = ''
				this.newProvider.clientId = ''
				this.newProvider.clientSecret = ''
				this.newProvider.discoveryEndpoint = ''
				this.showNewProvider = false
			} catch (error) {
				logger.error('Could not register a provider: ' + error.message, { error })
				showError(t('user_oidc', 'Could not register provider:') + ' ' + (error.response?.data?.message ?? error.message))
			}
		},
	},
}
</script>
<style lang="scss" scoped>
h2 .action-item {
	vertical-align: middle;
	margin-top: -2px;
}

h3 {
	font-weight: bold;
	padding-bottom: 12px;
}

.oidcproviders {
	margin-top: 20px;
	border-top: 1px solid var(--color-border);
	max-width: 900px;
}

.oidcproviders__provider {
	border-bottom: 1px solid var(--color-border);
	padding: 10px;
	display: flex;

	&:hover {
		background-color: var(--color-background-hover);
	}
	.oidcproviders__details {
		flex-grow: 1;
	}
}

.providermodal__wrapper {
	min-width: 320px;
	width: 50vw;
	max-width: 800px;
	height: calc(80vh - 40px);
	margin: 20px;
	overflow: scroll;
}
</style>
