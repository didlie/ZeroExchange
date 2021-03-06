<template>
	<div id="homeSearch" class="container">
		<div class="row">
	        <div class="col s12 m7 l9">
	        	<component :is="home_navbar" active="search" :user-info="userInfo" style="margin-bottom: 0;" class="z-depth-2"></component>
	        	<nav style="background-color: #1976D2; margin-bottom: 15px;">
		        	<div class="nav-wrapper">
		        		<form>
	    		        	<div class="input-field">
	    		        		<input id="search" type="search" placeholder="Search" v-model="searchInput" v-on:input="getQuestions" required>
	    		        		<label class="label-icon" for="search"><i class="material-icons">search</i></label>
	    		        		<i class="material-icons" v-on:click="clearSearch()">close</i>
	    			        </div>
	    		      	</form>
		        	</div>
		        </nav>

				<!--<ul class="pagination" v-if="questions.length != 0">
					<li><a href="#!" v-on:click.prevent="previousPage"><i class="material-icons">chevron_left</i></a></li>
					<li class="disabled"><a href="#!">{{ pageNum + 1 }}</a></li>
					<li><a href="#!" v-on:click.prevent="nextPage"><i class="material-icons">chevron_right</i></a></li>
				</ul>-->

		        <component :is="question_list_item" v-for="question in questions" :merger-zites="mergerZites" :user-info="userInfo" :question="question" :show-name="true" :show-topic-name="true" v-on:update="getQuestions"></component>

				<ul class="pagination" v-if="questions.length != 0">
					<li><a href="#!" v-on:click.prevent="previousPage"><i class="material-icons">chevron_left</i></a></li>
					<li class="disabled"><a href="#!">{{ pageNum + 1 }}</a></li>
					<li><a href="#!" v-on:click.prevent="nextPage"><i class="material-icons">chevron_right</i></a></li>
				</ul>
	        </div>
	        <div class="col s12 m5 l3">
	        	<component :is="connected_topics" :merger-zites="mergerZites"></component>
	        </div>
	    </div>
	</div>
</template>

<script>
	var moment = require("moment");
	var Router = require("../libs/router.js");
	var HomeNavbar = require("../vue_components/home_navbar.vue");
	var ConnectedTopics = require("../vue_components/connected_topics.vue");

	var QuestionListItem = require("../vue_components/question_list_item.vue");

	module.exports = {
		props: ["userInfo", "mergerZites"],
		name: "homeSearch",
		data: () => {
			return {
				home_navbar: HomeNavbar,
				connected_topics: ConnectedTopics,
				question_list_item: QuestionListItem,
				topicName: "",
				topicAddress: "",
				searchInput: "",
				pageNum: 0,
				questions: [],
				isSearchStrict: false // TODO
			}
		},
		beforeMount: function() {
			var self = this;

			if (!Router.currentParams["page"])
				this.page = 0;
			else this.page = parseInt(Router.currentParams["page"]) - 1;

			this.$parent.$on("update", function() {
				self.getQuestions();
			});

			this.$parent.$on("setMergerZites", function(mergerZites) {
				self.manageMerger(mergerZites);
				self.getQuestions();
			});

			// If mergerZites is empty
			if (this.mergerZites && Object.keys(this.mergerZites).length != 0 && this.mergerZites.constructor === Object) {
				this.manageMerger(this.mergerZites);
				this.getQuestions();
			}
		},
		computed: {
			mergerZiteValues: function() {
				if (this.mergerZites == null) {
					return [];
				}
				return Object.values(this.mergerZites);
			},
			mergerZiteKeys: function() {
				if (this.mergerZites == null) {
					return [];
				}
				return Object.keys(this.mergerZites);
			},
			getSearchResults: function() {
				var list = this.questions.slice();

				if (this.searchInput === "" || !this.searchInput) return list;

				var words = this.searchInput.trim().split(" ");
				var self = this;

				list = list.filter(function(question) {
					question.order = 0;
					var matches = 0;

					for (var i = 0; i < words.length; i++) {
                        var word = words[i].trim().toLowerCase();
                        
                        if ("solved".includes(word) && question.solution_id && question.solution_auth_address) {
                            question.order += 4;
                            matches++;
                            continue;
                        }

						if (question.tags && question.tags !== "" && question.tags.toLowerCase().includes(word)) {
							question.order += 4;
							matches++;
							continue;
						}
						if (question.title && question.title.toLowerCase().includes(word)) {
							question.order += 3;
							matches++;
							continue;
						}
						if (question.cert_user_id && word[0] === "@" && word.length > 1) {
							var wordId = word.substring(1, word.length);
							if (question.cert_user_id.replace(/@.*\.bit/, '').toLowerCase().includes(wordId)) {
								question.order += 2;
								matches++;
								continue;
							}
						}
						if (question.cert_user_id && question.cert_user_id.toLowerCase().includes(word)) {
							question.order += 2;
							matches++;
							continue;
                        }
                        if (question.site.toLowerCase().includes(word) || self.mergerZites[question.site].content.title.toLowerCase().includes(word)) {
							question.order += 2;
							matches++;
							continue;
						}
						if (question.body && question.body.toLowerCase().includes(word)) {
							matches++;
							continue;
						}

						if (self.isSearchStrict) {
							return false;
						}

						question.order--;
					}
					if (!self.isSearchStrict && matches === 0) return false;
					else return true;
				});
				if (list.length > 1) {
					list.sort(function(a, b) {
						return b.order - a.order;
					});
				}
				return list;
			}
		},
		methods: {
			manageMerger: function(mergerZites) {
			},
			getQuestions: function() {
				var self = this;

				page.getQuestionsSearch(this.searchInput, this.pageNum, 5)
					.then((questions) => {
						if (questions.length == 0 && self.pageNum != 0) {
							self.pageNum--;
							self.getQuestions();
							return;
						}
						self.questions = questions;
					});
			},
			goto: function(to) {
				Router.navigate(to);
			},
			getDate: function(date) {
				return moment(date).fromNow();
			},
			previousPage: function() { // TODO: Scroll to top
				this.pageNum -= 1;
				if (this.pageNum <= 0) this.pageNum = 0;
				this.getQuestions();
			},
			nextPage: function() {
				this.pageNum += 1;
				this.getQuestions();
			},
			clearSearch: function() {
				this.searchInput = "";
				this.getQuestions();
			}
		}
	}
</script>