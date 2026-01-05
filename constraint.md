<div class="se-contents" role="textbox" style="box-sizing: content-box; font-family: &quot;맑은 고딕&quot;; font-size: 16px; line-height: 1.6;" data-document-fixed-width="false">
	<p style="margin: 0px;"><span>*</span></p>
	<p style="margin: 0px;"><span> 논리적 복제의 제약 </span></p>
	<p style="margin: 0px;"><br></p>
	<p style="margin: 0px;"><span>결론 :&nbsp;</span></p>
	<p style="margin: 0px;"><span>DR용 복제의 목적 - 가장 최신의 Data상태로 두 Database를 일치시킨다 (물리적 복제)</span></p>
	<p style="margin: 0px;"><span>CDC 캡쳐의 목적 - Database의 변경이력을 추적하도록 제공한다 (논리적 복제)</span></p>
	<p style="margin: 0px;"><span>의 차이로 두 방식은 다름.</span></p>
	<p style="margin: 0px;"><br></p>
	<p style="margin: 0px;"><span>따라서,</span></p>
	<p style="margin: 0px;"><span style="text-decoration: underline; background-color: rgb(255, 255, 191);">CDC 캡쳐로 DR용 복제를 하는 경우,&nbsp;</span><span style="text-decoration: underline; background-color: rgb(255, 255, 191);">제약사항이 없어야만 사용할 수 있음.</span></p>
	<p style="margin: 0px;"><br></p>
	<p style="margin: 0px;"><span style="font-weight: 700;">* 대표적 제약</span></p>
	<p style="margin: 0px;"><span>&nbsp;&nbsp;- Schema, 권한 등&nbsp;</span><span style="text-decoration: underline;">Meta 정보 변경의 추적</span><span>은 제한적</span></p>
	<p style="margin: 0px;"><span>&nbsp;&nbsp;- LOB등&nbsp;</span><span style="text-decoration: underline;">Large 컬럼 정보의 완전한 추적</span><span>도 제한적</span></p>
	<p style="margin: 0px;"><br></p>
	<p style="margin: 0px;"><span style="font-weight: 700;">* PG의 경우,&nbsp;</span></p>
	<p style="margin: 0px;"><span>&nbsp;- 내장된&nbsp;</span><span style="background-color: rgb(255, 255, 191);">Log-shipping standby server</span><span>&nbsp;기능을 이용해서&nbsp;</span><span style="font-weight: 700;">물리적 복제</span><span>&nbsp;가능함.</span></p>
	<p style="margin: 0px 0px 0px 36px; display: block; overflow-wrap: break-word;"><span>관련 문서 ::&nbsp;</span><a style="text-decoration: underline; color: rgb(22, 63, 199);" data-href="https://www.postgresql.org/docs/current/warm-standby.html" data-hyperlink-id="se_8b55ca81-27f6-4fc5-9dfc-698a389c496c" target="_blank" href="https://www.postgresql.org/docs/current/warm-standby.html"><span>https://www.postgresql.org/docs/current/warm-standby.html</span></a></p>
	<p style="margin: 0px; display: block; overflow-wrap: break-word;"><span>&nbsp;-&nbsp;</span><span style="font-weight: 700;">논리적 복제</span><span>는 Debizium같은 솔루션 사용. (지금 개발중인 것에 해당)</span></p>
	<p style="margin: 0px;"><br></p>
	<p style="margin: 0px;"><br></p>
	<p style="margin: 0px;"><span style="text-decoration: underline; font-weight: 700;">* 침고: Gemini에 질문한 내용과 답변.</span></p>
	<p style="margin: 0px;"><span>&nbsp;&nbsp;- 용어 Unchanged TOAST (&nbsp;</span><span style="font-weight: 700;">T</span><span>he&nbsp;</span><span style="font-weight: 700;">O</span><span>versized&nbsp;</span><span style="font-weight: 700;">A</span><span>ttribute&nbsp;</span><span style="font-weight: 700;">S</span><span>torage&nbsp;</span><span style="font-weight: 700;">T</span><span>echnique)</span></p>
	<p style="margin: 0px;"><span>&nbsp;&nbsp; &nbsp;: Large 컬럼값이 바뀌지 않았을 때, 바뀌지 않았다는 marker</span></p>
	<p style="margin: 0px;"><span>&nbsp;&nbsp;- &nbsp;질문에서 물어본 제약사항</span></p>
	<p style="margin: 0px;"><span>&nbsp; &nbsp; &nbsp; &nbsp;Update/insert에서 이전 row 식별값으로 전달될 경우, 이전 Row를 완벽하게 특정할 방법 없음</span></p>
	<p style="margin: 0px;"><span>&nbsp;&nbsp; &nbsp; &nbsp;( spec상 제약사항 - 모든 상용솔루션 공통 )</span></p>
	<p style="margin: 0px;"><span>&nbsp;&nbsp; &nbsp; &nbsp;(&nbsp;</span><span style="text-decoration: underline; font-weight: 700;">Large 컬럼 + non-index 테이블 + update/delete</span><span style="text-decoration: underline; font-weight: 700;">&nbsp;인 경우에 한정한 유일한 제약</span><span>&nbsp;)</span></p>
	<p style="margin: 0px;"><br></p>
	<div class="se-para-div" style="margin: 0px;">
		<user-query class="ng-star-inserted" style="-webkit-box-orient: horizontal; -webkit-box-direction: normal; flex-direction: row; padding-inline-start: 0px; font-size: medium; font-variant-ligatures: normal; font-variant-caps: normal; orphans: 2; text-transform: none; widows: 2; -webkit-text-stroke-width: 0px; font-style: normal; font-family: &quot;Noto Sans KR&quot;; color: rgb(31, 31, 31); font-weight: 400; background-color: rgb(255, 255, 255); text-indent: 0px; padding: 8px 0px; word-spacing: 0px; width: 760px; max-width: 760px; display: block; white-space: normal;">
			<span _ngcontent-ng-c1317927767="" class="user-query-container" style="display: flex; width: 760px;">
				<user-query-content _ngcontent-ng-c1317927767="" class="user-query-container" _nghost-ng-c4058884339="" style="display: flex; -webkit-box-orient: horizontal; -webkit-box-direction: normal; flex-flow: row; padding-inline-start: 0px; --line-height: 28px; -webkit-box-pack: end; justify-content: flex-end; padding: 0px; width: 760px; --max-lines-for-collapse-count: 5;">
					<div _ngcontent-ng-c4058884339="" class="user-query-container" style="display: flex; -webkit-box-orient: vertical; -webkit-box-direction: normal; flex-direction: column; -webkit-box-pack: end; justify-content: flex-end; gap: 0px; margin: 0px; padding-bottom: 24px; min-width: 0px; user-select: none;">
						<div _ngcontent-ng-c4058884339="" class="query-content ng-star-inserted" id="user-query-content-10" jslog="275422;track:impression,attention" data-hveid="0" decode-data-ved="1" data-ved="0CAAQ3ucQahcKEwio1-DClfORAxUAAAAAHQAAAAAQUA" style="color: rgb(31, 31, 31); overflow-wrap: anywhere; min-width: 0px; user-select: text; display: flex; -webkit-box-orient: horizontal; -webkit-box-direction: normal; flex-direction: row; -webkit-box-pack: end; justify-content: flex-end; padding: 0px; margin-inline-start: 52px; margin-bottom: 8px;">
							<span _ngcontent-ng-c4058884339="" class="user-query-bubble-with-background ng-star-inserted" style="padding: 12px 16px; background: none 0% 0% / auto repeat scroll padding-box border-box rgb(233, 238, 246); border-radius: 24px 4px 24px 24px; max-width: 452px;">
								<span _ngcontent-ng-c4058884339="" class="horizontal-container" style="display: flex; -webkit-box-orient: horizontal; -webkit-box-direction: normal; flex-direction: row;">
									<div _ngcontent-ng-c4058884339="" role="heading" aria-level="2" class="query-text gds-body-l" dir="ltr" style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 16px; font-weight: 400; letter-spacing: normal; line-height: 28px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; font-variant-ligatures: none; margin-block: 0px; padding-top: 0px; overflow: hidden;">
										<p _ngcontent-ng-c4058884339="" class="query-text-line ng-star-inserted" style="margin: 0px;">Update, Delete에서 oldRow식별정보에 unchanged Toast 문제는 Debezium같은 솔루션에서도 기능제약사항이라고 했다. 그렇다면, PG에서 제공하는 Log-Shipping Standby Servers를 이용한 복제도 같은 문제가 있어야 하지 않나?</p>
									</div>
								</span>
							</span>
						</div>
					</div>
				</user-query-content>
			</span>
		</user-query>
	</div>
	<div class="se-para-div" style="margin: 0px;">
		<model-response class="ng-star-inserted" style="font-size: medium; font-variant-ligatures: normal; font-variant-caps: normal; orphans: 2; text-transform: none; widows: 2; -webkit-text-stroke-width: 0px; font-style: normal; font-family: &quot;Noto Sans KR&quot;; color: rgb(31, 31, 31); font-weight: 400; background-color: rgb(255, 255, 255); text-indent: 0px; word-spacing: 0px; width: 760px; display: block; white-space: normal;">
			<div _ngcontent-ng-c4130889771="">
				<response-container _ngcontent-ng-c4130889771="" _nghost-ng-c1105277494="" class="ng-tns-c1105277494-50 reduced-bottom-padding ng-star-inserted" jslog="188576;track:impression,attention;BardVeMetadataKey:[[&quot;r_e73156e2d72abbf1&quot;,&quot;c_74eda2e6c323120a&quot;,null,null,null,null,null,null,1,null,null,null,0]];mutable:true" data-hveid="0" decode-data-ved="1" data-ved="0CAAQoMELahcKEwio1-DClfORAxUAAAAAHQAAAAAQUQ">
					<div _ngcontent-ng-c1105277494="" class="response-container ng-tns-c1105277494-50 response-container-with-gpi ng-star-inserted response-container-has-multiple-responses" jslog="173900;track:impression,attention" data-hveid="1" style="background-color: rgb(255, 255, 255); border-radius: 16px; padding: 0px 0px 20px; display: flex; -webkit-box-orient: vertical; -webkit-box-direction: normal; flex-direction: column; position: relative; min-height: 100%; padding-inline: 0px;">
						<div _ngcontent-ng-c1105277494="" class="response-container-header ng-tns-c1105277494-50 ng-star-inserted" style="display: flex; flex-wrap: wrap; -webkit-box-align: center; align-items: center;">
							<div _ngcontent-ng-c1105277494="" class="response-container-header-controls ng-tns-c1105277494-50" style="-webkit-box-flex: 1; flex: 1 1 0%; min-width: 100%;">
								<tts-control _ngcontent-ng-c1105277494="" _nghost-ng-c580904282="" class="ng-tns-c1105277494-50 ng-trigger ng-trigger-singleResponseEnter ng-animate-disabled ng-star-inserted">
									<div _ngcontent-ng-c580904282="" class="response-tts-container ng-star-inserted" data-test-draft-id="rc_e2f73cc1561db2c2" style="position: absolute; top: 4px; inset-inline-end: 0px; opacity: 0; transition: opacity 0.2s linear; pointer-events: none; height: 1827px;">
										<div _ngcontent-ng-c580904282="" data-test-id="disabled-tooltip" class="mat-mdc-tooltip-trigger tts-button-container mat-mdc-tooltip-disabled">
											<button _ngcontent-ng-c580904282="" mat-icon-button="" class="mdc-icon-button mat-mdc-icon-button mat-mdc-button-base mat-mdc-tooltip-trigger tts-button mat-unthemed ng-star-inserted" mat-ripple-loader-uninitialized="" mat-ripple-loader-class-name="mat-mdc-button-ripple" mat-ripple-loader-centered="" aria-label="듣기" jslog="184512;track:generic_click,impression;BardVeMetadataKey:[[null,null,null,&quot;rc_e2f73cc1561db2c2&quot;,null,null,&quot;ko&quot;,null,1,null,null,1,0]];mutable:true" style="--mat-focus-indicator-border-radius: 50%; user-select: none; display: inline-block; position: relative; box-sizing: border-box; border: none; outline: none; background-color: rgba(0, 0, 0, 0); fill: currentcolor; text-decoration: none; cursor: pointer; z-index: 0; overflow: visible; border-radius: 9999px; flex-shrink: 0; text-align: center; width: 40px; height: 40px; padding: 8px; font-size: 24px; color: rgb(68, 71, 70); -webkit-tap-highlight-color: rgba(0, 0, 0, 0); --mdc-icon-button-state-layer-size: 40px; --mat-icon-button-state-layer-size: 40px; margin-inline-end: -10px;"></button>
										</div>
									</div>
									<div _ngcontent-ng-c580904282="" role="menu" class="mat-mdc-menu-trigger playback-speed-menu-trigger multi" aria-haspopup="menu" aria-expanded="false" style="position: absolute; inset-inline-end: 46px; top: 46px;"></div>
								</tts-control>
							</div>
							<div _ngcontent-ng-c1105277494="" class="response-container-header-status ng-tns-c1105277494-50" style="display: flex; -webkit-box-orient: vertical; -webkit-box-direction: normal; flex-direction: column; -webkit-box-align: start; align-items: flex-start; gap: 4px; align-self: flex-start; min-width: 0px; -webkit-box-flex: 1; flex: 1 1 0%;">
								<div _ngcontent-ng-c1105277494="" class="response-container-header-processing-state ng-tns-c1105277494-50"></div>
							</div>
						</div>
						<div _ngcontent-ng-c1105277494="" class="presented-response-container ng-tns-c1105277494-50" jslog="283401;track:impression,attention" data-hveid="2" style="position: relative; display: flex; -webkit-box-orient: horizontal; -webkit-box-direction: normal; flex-direction: row; min-height: 100%;">
							<div _ngcontent-ng-c1105277494="" class="avatar-gutter ng-tns-c1105277494-50 ng-star-inserted" style="display: flex; -webkit-box-orient: vertical; -webkit-box-direction: normal; flex-direction: column;">
								<bard-avatar _ngcontent-ng-c1105277494="" class="avatar-component ng-tns-c1693567158-51 ng-tns-c1105277494-50 ng-star-inserted" _nghost-ng-c1693567158="" style="margin-inline-end: 20px; -webkit-box-flex: 0; flex-grow: 0; flex-shrink: 0;">
									<div _ngcontent-ng-c1693567158="" bardavataranimationscontroller="" class="bard-avatar ng-tns-c1693567158-51" style="display: flex; min-height: 32px; width: 32px; position: relative; margin-inline-start: 0px;">
										<div _ngcontent-ng-c1693567158="" class="avatar-container ng-tns-c1693567158-51" style="-webkit-box-orient: vertical; -webkit-box-direction: normal; flex-direction: column; transform: scale(1);">
											<div _ngcontent-ng-c1693567158="" class="avatar avatar_primary ng-tns-c1693567158-51 ng-star-inserted" style="height: 32px; width: 32px; display: flex; -webkit-box-align: center; align-items: center; -webkit-box-pack: center; justify-content: center; user-select: none;">
												<div _ngcontent-ng-c1693567158="" class="avatar_primary_model is-gpi-avatar ng-tns-c1693567158-51" style="height: 32px; width: 32px; display: flex; -webkit-box-align: unset; align-items: unset; -webkit-box-pack: center; justify-content: center; pointer-events: none;">
													<div _ngcontent-ng-c1693567158="" lottie-animation="" class="avatar_primary_animation is-gpi-avatar aurora-enabled ng-tns-c1693567158-51" data-test-lottie-animation-status="completed" style="width: 32px; height: 32px;">
														<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" viewBox="0 0 32 32" width="32" height="32" preserveAspectRatio="xMidYMid meet" style="width: 100%; height: 100%; transform: translate3d(0px, 0px, 0px); content-visibility: visible;"><defs><linearGradient id="__lottie_element_951" spreadMethod="pad" gradientUnits="userSpaceOnUse" x1="-33" y1="26" x2="31" y2="-28"></linearGradient><linearGradient id="__lottie_element_955" spreadMethod="pad" gradientUnits="userSpaceOnUse" x1="-33" y1="26" x2="31" y2="-28"></linearGradient></defs></svg>
													</div>
												</div>
											</div>
										</div>
										<div _ngcontent-ng-c1693567158="" lottie-animation="" class="avatar_spinner_animation ng-tns-c1693567158-51" style="position: absolute; top: 0px; width: 32px; height: 32px; pointer-events: none; opacity: 0; visibility: hidden; transition: opacity 0.1s 0.1s; transform: scale(1);">
											<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" viewBox="0 0 32 32" width="32" height="32" preserveAspectRatio="xMidYMid meet" style="width: 100%; height: 100%; transform: translate3d(0px, 0px, 0px); content-visibility: visible;"><defs><clipPath id="__lottie_element_956"><rect width="32" height="32" x="0" y="0"></rect></clipPath><linearGradient id="__lottie_element_960" spreadMethod="pad" gradientUnits="userSpaceOnUse"></linearGradient><linearGradient id="__lottie_element_967" spreadMethod="pad" gradientUnits="userSpaceOnUse"></linearGradient><linearGradient id="__lottie_element_971" spreadMethod="pad" gradientUnits="userSpaceOnUse"></linearGradient><linearGradient id="__lottie_element_975" spreadMethod="pad" gradientUnits="userSpaceOnUse" x1="0" y1="-16" x2="16" y2="0"></linearGradient><linearGradient id="__lottie_element_979" spreadMethod="pad" gradientUnits="userSpaceOnUse" x1="0" y1="-16" x2="0" y2="16"></linearGradient><linearGradient id="__lottie_element_983" spreadMethod="pad" gradientUnits="userSpaceOnUse" x1="-16" y1="0" x2="16" y2="0"></linearGradient></defs><g clip-path="url(#__lottie_element_956)"><g transform="matrix(1,0,0,1,16,16)" opacity="1" style="display: block;"><g opacity="1" transform="matrix(1,0,0,1,0,0)"><path stroke="url(#__lottie_element_983)" stroke-linecap="round" stroke-linejoin="miter" fill-opacity="0" stroke-miterlimit="4" stroke-opacity="1" stroke-width="2" d=" M-4.383999824523926,-14.347999572753906 C-2.996999979019165,-14.772000312805176 -1.524999976158142,-15 0,-15 C0,-15 0,-15 0,-15 C1.7100000381469727,-15 3.3540000915527344,-14.713000297546387 4.886000156402588,-14.1850004196167"></path></g></g></g></svg>
										</div>
									</div>
								</bard-avatar>
							</div>
							<div _ngcontent-ng-c1105277494="" class="response-container-content ng-tns-c1105277494-50" style="padding-top: 4px; -webkit-box-flex: 1; flex: 1 1 0%; height: auto; min-width: 0px; min-height: 100%;">
								<div _ngcontent-ng-c4130889771="" class="response-content ng-tns-c1105277494-50">
									<div _ngcontent-ng-c4130889771="" class="thoughts-container" style="position: relative;"></div>
									<structured-content-container _ngcontent-ng-c4130889771="" class="model-response-text ng-star-inserted" _nghost-ng-c908870705="" style="font-variant-ligatures: none; display: block; overflow-x: visible; width: 708px; color: rgb(31, 31, 31); height: auto;">
										<div _ngcontent-ng-c908870705="" class="container" style="padding: 8px;">
											<message-content _ngcontent-ng-c908870705="" _nghost-ng-c1582008977="" id="message-content-id-r_e73156e2d72abbf1" class="ng-star-inserted">
												<div _ngcontent-ng-c1582008977="" inline-copy-host="" class="markdown markdown-main-panel stronger enable-updated-hr-color" id="model-response-message-contentr_e73156e2d72abbf1" aria-live="polite" aria-busy="false" dir="ltr" style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 16px; font-weight: 400; letter-spacing: normal; line-height: 1.75rem; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; color: rgb(31, 31, 31); overflow-wrap: break-word; margin-inline-end: 0px; --animation-duration: 400ms; --fade-animation-function: linear;">
													<p data-path-to-node="0" style="margin: 0px 0px 16px; overflow-wrap: anywhere;">날카로운 통찰력입니다! 하지만 결론부터 말씀드리면,<span>&nbsp;</span><b data-path-to-node="0" data-index-in-node="29">Log-Shipping 기반의 Standby Server 복제에서는 이 문제가 전혀 발생하지 않습니다.</b></p>
													<p data-path-to-node="1" style="margin: 0px 0px 16px; overflow-wrap: anywhere;">그 이유는 **Logical Replication(CDC)**과 **Physical Replication(Log-Shipping)**이 로그를 읽고 해석하는 방식이 근본적으로 다르기 때문입니다.</p>
													<hr data-path-to-node="2" style="border-right: none; border-bottom: none; border-left: none; border-image: initial; border-top: 0.8px solid rgb(196, 199, 197);">
													<h2 data-path-to-node="3" style="margin: 28px 0px 8px; font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 20px; font-weight: 700; letter-spacing: normal; line-height: 28px; --gds-type-scale-default-wdth: &quot;wdth&quot; 100; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 100;">1. 물리적 복제(Log-Shipping) vs 논리적 복제(CDC)</h2>
													<p data-path-to-node="4" style="margin: 0px 0px 16px; overflow-wrap: anywhere;">Standby Server가 사용하는 방식과 Debezium이 사용하는 방식의 차이를 이해하면 답이 명확해집니다.</p>
													<h3 data-path-to-node="5" style="margin: 28px 0px 8px; font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 16px; font-weight: 700; letter-spacing: normal; line-height: 24px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95;">A. Log-Shipping (물리적 복제)</h3>
													<ul data-path-to-node="6" style="list-style-type: disc; padding-inline-start: 1.6875rem; margin: 8px 0px 16px;">
														<li style="margin: 8px 0px; padding-inline-start: 4px;">
															<p data-path-to-node="6,0,0" style="margin: 8px 0px; overflow-wrap: anywhere; padding-inline-start: 4px;"><b data-path-to-node="6,0,0" data-index-in-node="0">복제 대상:</b><span>&nbsp;</span>데이터의 "의미"가 아니라 **"디스크 블록의 변화(Binary)"**를 그대로 복제합니다.</p>
														</li>
														<li style="margin: 8px 0px; padding-inline-start: 4px;">
															<p data-path-to-node="6,1,0" style="margin: 8px 0px; overflow-wrap: anywhere; padding-inline-start: 4px;"><b data-path-to-node="6,1,0" data-index-in-node="0">동작 원리:</b><span>&nbsp;</span>WAL(Write Ahead Log)에 기록된 "X번 파일의 Y번 페이지의 Z바이트를 이렇게 바꿔라"라는 저수준 명령을 그대로 실행합니다.</p>
														</li>
														<li style="margin: 8px 0px; padding-inline-start: 4px;">
															<p data-path-to-node="6,2,0" style="margin: 8px 0px; overflow-wrap: anywhere; padding-inline-start: 4px;"><b data-path-to-node="6,2,0" data-index-in-node="0">TOAST 처리:</b><span>&nbsp;</span>TOAST 데이터가 별도의 테이블에 저장되어 있든 말든, 그 데이터가 담긴<span>&nbsp;</span><b data-path-to-node="6,2,0" data-index-in-node="52">물리적 블록 자체를 복사</b>하므로 'Unchanged'라는 개념 자체가 필요 없습니다. 그냥 소스 서버의 디스크와 타겟 서버의 디스크를 동기화하는 개념입니다.</p>
														</li>
													</ul>
													<h3 data-path-to-node="7" style="margin: 28px 0px 8px; font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 16px; font-weight: 700; letter-spacing: normal; line-height: 24px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95;">B. Logical Replication (Debezium 등 CDC)</h3>
													<ul data-path-to-node="8" style="list-style-type: disc; padding-inline-start: 1.6875rem; margin: 8px 0px 16px;">
														<li style="margin: 8px 0px; padding-inline-start: 4px;">
															<p data-path-to-node="8,0,0" style="margin: 8px 0px; overflow-wrap: anywhere; padding-inline-start: 4px;"><b data-path-to-node="8,0,0" data-index-in-node="0">복제 대상:</b><span>&nbsp;</span>**"어떤 테이블의 어떤 행이 어떻게 바뀌었는가"**라는 데이터의 논리적 변경 사항을 복제합니다.</p>
														</li>
														<li style="margin: 8px 0px; padding-inline-start: 4px;">
															<p data-path-to-node="8,1,0" style="margin: 8px 0px; overflow-wrap: anywhere; padding-inline-start: 4px;"><b data-path-to-node="8,1,0" data-index-in-node="0">동작 원리:</b><span>&nbsp;</span>WAL 로그를 해석(Decoding)하여<span>&nbsp;</span><code data-path-to-node="8,1,0" data-index-in-node="30" style="font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; font-family: &quot;Google Sans Mono&quot;, monospace; color: rgb(68, 71, 70); background-color: rgb(233, 238, 246); border-radius: 6px; padding: 1px 6px;">UPDATE table SET col = 'new'</code>와 같은 논리적 메시지로 변환합니다.</p>
														</li>
														<li style="margin: 8px 0px; padding-inline-start: 4px;">
															<p data-path-to-node="8,2,0" style="margin: 8px 0px; overflow-wrap: anywhere; padding-inline-start: 4px;"><b data-path-to-node="8,2,0" data-index-in-node="0">TOAST 처리:</b><span>&nbsp;</span>네트워크 비용을 아끼기 위해 "이 컬럼(TOAST)은 안 바뀌었으니 이번 메시지에는 안 보낼게"라는 **최적화(<code data-path-to-node="8,2,0" data-index-in-node="72" style="font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; font-family: &quot;Google Sans Mono&quot;, monospace; color: rgb(68, 71, 70); background-color: rgb(233, 238, 246); border-radius: 6px; padding: 1px 6px;">unchanged toast</code>)**가 들어갑니다. 바로 이 지점에서 '기능 제약'이나 '식별 문제'가 발생하는 것입니다.</p>
														</li>
													</ul>
													<hr data-path-to-node="9" style="border-right: none; border-bottom: none; border-left: none; border-image: initial; border-top: 0.8px solid rgb(196, 199, 197);">
													<h2 data-path-to-node="10" style="margin: 28px 0px 8px; font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 20px; font-weight: 700; letter-spacing: normal; line-height: 28px; --gds-type-scale-default-wdth: &quot;wdth&quot; 100; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 100;">2. 왜 CDC에서만 문제가 되는가?</h2>
													<p data-path-to-node="11" style="margin: 0px 0px 8px; overflow-wrap: anywhere;">Standby Server는 소스 DB의 **전체 데이터 파일(Heap, Index, TOAST 등)**을 그대로 가지고 있는 "복제판"입니다. 하지만 CDC의 목적은 보통 **"서로 다른 DB(예: PG to MySQL)"**나 **"서로 다른 구조"**로 데이터를 보내는 것입니다.</p>
													<ul data-path-to-node="12" style="list-style-type: disc; padding-inline-start: 1.6875rem; margin: 8px 0px 16px;">
														<li style="margin: 8px 0px; padding-inline-start: 4px;">
															<p data-path-to-node="12,0,0" style="margin: 8px 0px; overflow-wrap: anywhere; padding-inline-start: 4px;"><b data-path-to-node="12,0,0" data-index-in-node="0">Standby Server:</b><span>&nbsp;</span>이미 TOAST 테이블에 데이터가 다 들어 있으므로, 포인터만 복제되어도 데이터를 찾아가는 데 문제가 없습니다.</p>
														</li>
														<li style="margin: 8px 0px; padding-inline-start: 4px;">
															<p data-path-to-node="12,1,0" style="margin: 8px 0px; overflow-wrap: anywhere; padding-inline-start: 4px;"><b data-path-to-node="12,1,0" data-index-in-node="0">CDC (Debezium):</b><span>&nbsp;</span>타겟 DB(예: Kafka, BigQuery)에는 소스 DB의 TOAST 테이블이 없습니다. 따라서 소스에서 "안 바뀌었으니 포인터만 봐"라고 하면, 타겟 입장에서는 **"그 포인터가 가리키는 실제 데이터가 나한테는 없는데?"**라는 상황이 되는 것입니다.</p>
														</li>
													</ul>
													<hr data-path-to-node="13" style="border-right: none; border-bottom: none; border-left: none; border-image: initial; border-top: 0.8px solid rgb(196, 199, 197);">
													<h2 data-path-to-node="14" style="margin: 28px 0px 8px; font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 20px; font-weight: 700; letter-spacing: normal; line-height: 28px; --gds-type-scale-default-wdth: &quot;wdth&quot; 100; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 100;">3. 요약 비교</h2>
													<div class="horizontal-scroll-wrapper" style="overflow-x: scroll;">
														<div class="table-block-component" style="overflow: scroll hidden;">
															<response-element class="" ng-version="0.0.0-PLACEHOLDER">
																<table-block _nghost-ng-c3726268734="" class="ng-star-inserted">
																	<div _ngcontent-ng-c3726268734="" class="table-block has-export-button" style="width: fit-content; max-width: 100%; overflow: hidden; min-width: 0px;">
																		<div _ngcontent-ng-c3726268734="" not-end-of-paragraph="" class="table-content not-end-of-paragraph" jslog="275421;track:impression,attention" data-hveid="0" decode-data-ved="1" data-ved="0CAAQ3ecQahcKEwio1-DClfORAxUAAAAAHQAAAAAQWA" style="color: rgb(31, 31, 31); font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 16px; font-weight: 400; letter-spacing: normal; line-height: 28px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; overflow-x: auto;">
																			<table data-path-to-node="15" style="background-color: rgb(240, 244, 249); border-radius: 16px 16px 4px 4px; border-spacing: 0px; border-collapse: separate; width: 590.45px; overflow: hidden; border: none;">
																				<thead>
																					<tr>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,0,0,0">구분</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,0,1,0">Log-Shipping (Standby)</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,0,2,0">Logical Replication (CDC)</span>
																						</td>
																					</tr>
																				</thead>
																				<tbody>
																					<tr>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,1,0,0">
																								<b data-path-to-node="15,1,0,0" data-index-in-node="0">복제 단위</b>
																							</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,1,1,0">물리적 블록 (Physical Block)</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,1,2,0">논리적 로우 (Logical Row)</span>
																						</td>
																					</tr>
																					<tr>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,2,0,0">
																								<b data-path-to-node="15,2,0,0" data-index-in-node="0">TOAST 처리</b>
																							</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,2,1,0">블록 자체를 복사 (문제 없음)</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,2,2,0">변경 시에만 본문 전송 (Unchanged 발생)</span>
																						</td>
																					</tr>
																					<tr>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,3,0,0">
																								<b data-path-to-node="15,3,0,0" data-index-in-node="0">Old Row 식별</b>
																							</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,3,1,0">불필요 (파일 주소로 직접 제어)</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,3,2,0">필요 (PK나 모든 컬럼 비교 방식)</span>
																						</td>
																					</tr>
																					<tr>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,4,0,0">
																								<b data-path-to-node="15,4,0,0" data-index-in-node="0">제약 사항</b>
																							</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,4,1,0">버전/OS/아키텍처가 동일해야 함</span>
																						</td>
																						<td style="font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 14px; font-weight: 400; letter-spacing: normal; line-height: 20px; --gds-type-scale-default-wdth: &quot;wdth&quot; 95; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 95; background-color: rgb(248, 250, 253); padding: 8px 12px; vertical-align: top;">
																							<span data-path-to-node="15,4,2,0">이기종 간 복제 가능, 특정 타입 제약 발생</span>
																						</td>
																					</tr>
																				</tbody>
																			</table>
																		</div>
																		<div _ngcontent-ng-c3726268734="" hide-from-message-actions="" class="table-footer hide-from-message-actions ng-star-inserted" style="display: flex; -webkit-box-align: center; align-items: center; min-height: 52px; background-color: rgb(248, 250, 253); border-radius: 4px 4px 16px 16px; -webkit-box-pack: start; justify-content: flex-start; padding: 4px; margin-top: 4px; gap: 0px;"></div>
																	</div>
																</table-block>
															</response-element>
														</div>
													</div>
													<hr data-path-to-node="16" style="border-right: none; border-bottom: none; border-left: none; border-image: initial; border-top: 0.8px solid rgb(196, 199, 197);">
													<h2 data-path-to-node="17" style="margin: 28px 0px 8px; font-family: &quot;Google Sans Flex&quot;, &quot;Google Sans&quot;, &quot;Helvetica Neue&quot;, sans-serif; font-size: 20px; font-weight: 700; letter-spacing: normal; line-height: 28px; --gds-type-scale-default-wdth: &quot;wdth&quot; 100; font-variation-settings: &quot;ROND&quot; 0, &quot;slnt&quot; 0, &quot;wdth&quot; 100;">4. 결론</h2>
													<p data-path-to-node="18" style="margin: 0px 0px 16px; overflow-wrap: anywhere;">질문하신 **"Standby Server도 같은 문제가 있어야 하는 것 아닌가?"**에 대한 대답은 **"아니오"**입니다. Standby Server는 소스 DB의 모든 것을 물리적으로 똑같이 복사해서 들고 있기 때문에, TOAST 데이터가 어디 있는지 이미 알고 있기 때문입니다.</p>
													<p data-path-to-node="19" style="margin: 0px 0px 16px; overflow-wrap: anywhere;">반면,<span>&nbsp;</span><b data-path-to-node="19" data-index-in-node="4">Debezium 같은 CDC 도구</b>는 소스 DB의 물리적 구조를 복제하는 것이 아니라 **"변경 사건(Event)"**만 전달받기 때문에, 전달되지 않은 과거 데이터(Unchanged TOAST)에 대해서는 알 길이 없어 제약이 생기는 것입니다.</p>
													<p data-path-to-node="20" style="margin: 0px 0px 16px; overflow-wrap: anywhere;">이제 직접 만드시는 프로그램이 왜 Debezium과 같은 고민을 해야 하는지, 그리고 왜 Standby Server 방식으로는 그 고민이 해결되지 않는지 정리가 되셨을까요?</p>
												</div>
											</message-content>
										</div>
									</structured-content-container>
								</div>
							</div>
						</div>
					</div>
				</response-container>
			</div>
		</model-response>
	</div>
</div>
