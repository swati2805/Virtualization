# Virtualization
README
Assignment 1 CMPE 283

Team Memebers:

Akhil Devarshetty(013820586)
Swati Shukla (013860366)


Linux kernel
============

There are several guides for kernel developers and users. These guides can
be rendered in a number of formats, like HTML and PDF. Please read
Documentation/admin-guide/README.rst first.

In order to build the documentation, use ``make htmldocs`` or
``make pdfdocs``.  The formatted documentation can also be read online at:

    https://www.kernel.org/doc/html/latest/

There are various text files in the Documentation/ subdirectory,
several of them using the Restructured Text markup notation.

Please read the Documentation/process/changes.rst file, as it contains the
requirements for building and running the kernel, and information about
the problems which may result by upgrading your kernel.

===============================================================================================================================
Steps Used To Build And Complete This Assignment :
=========================

PreRequirement: Use a virtualization enabled machine with atleast 200 GB or more space.

Step 1: First of all, Get VmWare Fusion setup installed.

Step 2: Then install Linux Ubuntu on VMware Fusion. 

Step 3: Then install GitHub in our laptop.

Step 4: Then fork the Linux Repo in your GitHub Account.

Step 5: Then pull the Linux Git Repo into your Ubuntu Linux.

Step 6: Then run the following CL commands:
 	6.1 Sudo apt-get install libncurses-dev

	6.2 Sudo apt-get install libssl-dev

	6.3 Make menuconfig

	6.4 Make

	6.5 Make modules

	6.6 Make modules_install

	6.7 Make install

	6.8 Reboot

Step 7: Then Install kvm as follows:

	7.1 sudo apt install cpu-checker

	7.2 sudo kvm-ok

	7.2 sudo apt update

	7.3 sudo apt install qemu qemu-kvm libvirt-bin  bridge-utils  virt-manager

	7.4 sudo service libvirtd start

	7.5 sudo update-rc.d libvirtd enable

	7.6 service libvirtd status

	7.7 sudo virt-manager

Step 8: Then get inside Virt-Manager and again install Ubuntu kvm following the same steps as Step 7 and build the Kernel Code in next steps.

Step 9: Then Scp the 283-1c file and makefile. 

         9.1 Run the files using the commands as follows: 

	 9.1.1 Make all

	 9.1.2 ./insmod cmpe283.ko

	 9.1.3	Dimesg

Step 10: Now you will arrive at your final steps of making Hypercall to CMPE283 by following these steps: 
  
	10.1 CD / Goto the following path:  /lnux/arch/x86/kvm/x86.c file in the Host System

	10.2 Inside the kvm_emulate_hypercall() add this code as one of the Case and recompile building the kernel:

		Case 0x283:
		ret = 0x0033383245504D43;

		break;

Step 11: Run the test code given in Canvas App provided in discussion thread of CMPE283 Fall 2019 Subject and follow :
`	11.1. Copy the .tar.gz file containing the test code to the inner vm.
	11.2. Extract the tar.gz file.
	11.3  Implement the following command : sudo apt install gcc make.
	11.4. CD to 283
	11.5. The do "make" 
	11.6. Then, Implement the command : sudo insmod cmpe283.ko
	11.7. dmesg
	11.8  Reboot

 => This will result in answer of Hypercall as 0x0033383245504D43 in your system. You are communicating with the Hypervisor henceforth.
________________________________________________________________________________________________________________________________________

Assignment 2: Instrumentation Via Hypercall: For Instruction 1 and Instruction 3: 
________________________________________________________________________________________________________________________________________
Steps Used to Complete The Assignment:		
Prerequirement: Use a virtualization enabled machine with at least 200 GB or more space capable of running Linux.
     	Steps:
	Step 1: Ensure your system is ready to the point of setup till Assignment 1.
	Step 2: Build the new linux kernel module.
sudo apt-get install libncurses-dev 
sudo apt-get install libssl-dev - make menuconfig // Save the config file.
 make && make modules && make modules_install && make install // Builds Linux Kernel 
	Step 3: Add code to the file at arch/x86/kvm/ : 
       -	Below is the code we modified of cpuid.c at arch/x86/kvm/cpuid.c  :	




int kvm_emulate_cpuid(struct kvm_vcpu *vcpu)




{


	


	u32 eax, ebx, ecx, edx;







	if (cpuid_fault_enabled(vcpu) && !kvm_require_cpl(vcpu, 0))


		return 1;







	eax = kvm_rax_read(vcpu);


	ecx = kvm_rcx_read(vcpu);


	if(eax  ==  0x4fffffff)


	{


		


	    eax = exits;


		


	}


	else if(eax  ==  0x4ffffffd)


	{


		


       if(ecx >= 0 && ecx < 62)


	{


		eax = exits_per_reason[(int)ecx];


	}


          


	}


	else


	{


		


	kvm_cpuid(vcpu, &eax, &ebx, &ecx, &edx, true);


	


	}


	kvm_rax_write(vcpu, eax);


	kvm_rbx_write(vcpu, ebx);


	kvm_rcx_write(vcpu, ecx);


	kvm_rdx_write(vcpu, edx);


	return kvm_skip_emulated_instruction(vcpu);


}







/*


Function to get the exit reasons


*/







void add_exit_per_reason(u32 exit_reason)


{







   if(exit_reason >= 0 && exit_reason < 62)


   {


	   


       exits++;


       exits_per_reason[(int)exit_reason]++;


  


   }


	


}





Below is the code we modified of vmx.c arch/x86/kvm/vmx/vmx.c  :
void add_exit_per_reason(u32 exit_reason);

/*
 * The guest has exited.  See if we can fix it or if we need userspace
 * assistance.
 */
static int vmx_handle_exit(struct kvm_vcpu *vcpu)
{
	struct vcpu_vmx *vmx = to_vmx(vcpu);
	u32 exit_reason = vmx->exit_reason;
	u32 vectoring_info = vmx->idt_vectoring_info;
    
    add_exit_per_reason(exit_reason);

	trace_kvm_exit(exit_reason, vcpu, KVM_ISA_VMX);

	/*
	 * Flush logged GPAs PML buffer, this will make dirty_bitmap more
	 * updated. Another good is, in kvm_vm_ioctl_get_dirty_log, before
	 * querying dirty_bitmap, we only need to kick all vcpus out of guest
	 * mode as if vcpus is in root mode, the PML buffer must has been
	 * flushed already.
	 */
	if (enable_pml)
		vmx_flush_pml_buffer(vcpu);

	/* If guest state is invalid, start emulating */
	if (vmx->emulation_required)
		return handle_invalid_guest_state(vcpu);

	if (is_guest_mode(vcpu) && nested_vmx_exit_reflected(vcpu, exit_reason))
		return nested_vmx_reflect_vmexit(vcpu, exit_reason);

	if (exit_reason & VMX_EXIT_REASONS_FAILED_VMENTRY) {
		dump_vmcs();
		vcpu->run->exit_reason = KVM_EXIT_FAIL_ENTRY;
		vcpu->run->fail_entry.hardware_entry_failure_reason
			= exit_reason;
		return 0;
	}
	Step 4: Build the updated code :
	-make modules 
-rmmod kvm 
-rmmod kvm_intel 
Step 5: Install virt-manager. Download Ubuntu ISO file and create test VM. 
- sudo apt-get install qemu-kvm libvirt-bin bridge-utils virt-manager 
- sudo apt-get install virt-manager 
Step 6: Open Virtual-Manager and start Virtual Machine. Now try dmesg command in the host system’s terminal.
 - dmesg 
 ____________________________________________________________________________________________________________________________________________
 Assignment 3: Instrumentation Via Hypercall: For Instruction 2 and Instruction 4:
 ______________________________________________________________________________________________________________________________
 Steps Used to Complete The Assignment:		
Prerequirement: Use a virtualization enabled machine with at least 200 GB or more space capable of running Linux.
     	Steps:
	Step 1: Ensure your system is ready to the point of setup till Assignment 2.
	Step 2: Build the new linux kernel module.
sudo apt-get install libncurses-dev 
sudo apt-get install libssl-dev - make menuconfig // Save the config file.
 make && make modules && make modules_install && make install // Builds Linux Kernel 
	Step 3: Add code to the file at arch/x86/kvm/ : 
       -	Below is the code we modified of cpuid.c at arch/x86/kvm/cpuid.c  :







static atomic_t exits,exits_per_reason[62];








static atomic64_t exits_time,exits_time_per_reason[62];







void add_exit_time_per_reason(u32 exit_reason,u64 time_taken);














/*atomic_t exits;




EXPORT_SYMBOL(exits);*/

if(eax  ==  0x4fffffff){








	    eax = atomic_read(&exits);




	}else if(eax  ==  0x4ffffffe){




       ebx = ( (atomic64_read(&exits_time) >> 32) );




		ecx = ( (atomic64_read(&exits_time) & 0xFFFFFFFF ));	   




   }else if(eax  ==  0x4ffffffd){




       if(ecx >= 0 && ecx < 62)	   




           eax = atomic_read(&exits_per_reason[(int)ecx]);




	}else if(eax  ==  0x4ffffffc){




       if(ecx >= 0 && ecx < 62){       




           ebx = ( (atomic64_read(&exits_time_per_reason[(int)ecx]) >> 32) );




		    ecx = ( (atomic64_read(&exits_time_per_reason[(int)ecx]) & 0xFFFFFFFF ));




       }	   




   }else{




	    kvm_cpuid(vcpu, &eax, &ebx, &ecx, &edx, true);




	}




	kvm_rax_write(vcpu, eax);




	kvm_rbx_write(vcpu, ebx);


@@ -1084,22 +1078,15 @@ int kvm_emulate_cpuid(struct kvm_vcpu *vcpu)




	return kvm_skip_emulated_instruction(vcpu);




}

void add_exit_time_per_reason(u32 exit_reason,u64 time_taken){








   if(exit_reason >= 0 && exit_reason < 62){   




      atomic64_add(time_taken,&exits_time);




       atomic64_add(time_taken,&exits_time_per_reason[(int)exit_reason]);




       atomic_inc(&exits);




      atomic_inc(&exits_per_reason[(int)exit_reason]);




   }

EXPORT_SYMBOL_GPL(kvm_emulate_cpuid);








EXPORT_SYMBOL_GPL(add_exit_per_reason);




EXPORT_SYMBOL_GPL(add_exit_time_per_reason);

-   Below is the code we modified of vmx.c arch/x86/kvm/vmx/vmx.c  :


static void vmx_exit(void)








{




#ifdef CONFIG_KEXEC_CORE




	RCU_INIT_POINTER(crash_vmclear_loaded_vmcss, NULL);




	synchronize_rcu();




#endif










	kvm_exit();










#if IS_ENABLED(CONFIG_HYPERV)




	if (static_branch_unlikely(&enable_evmcs)) {




		int cpu;




		struct hv_vp_assist_page *vp_ap;




		/*




		 * Reset everything to support using non-enlightened VMCS




		 * access later (e.g. when we reload the module with




		 * enlightened_vmcs=0)




		 */




		for_each_online_cpu(cpu) {




			vp_ap =	hv_get_vp_assist_page(cpu);










			if (!vp_ap)




				continue;










			vp_ap->nested_control.features.directhypercall = 0;




			vp_ap->current_nested_vmcs = 0;




			vp_ap->enlighten_vmentry = 0;




		}










		static_branch_disable(&enable_evmcs);




	}




#endif




	vmx_cleanup_l1d_flush();




}




module_exit(vmx_exit);










static int __init vmx_init(void)




{




	int r;










#if IS_ENABLED(CONFIG_HYPERV)




	/*




	 * Enlightened VMCS usage should be recommended and the host needs




	 * to support eVMCS v1 or above. We can also disable eVMCS support




	 * with module parameter.




	 */




	if (enlightened_vmcs &&




	    ms_hyperv.hints & HV_X64_ENLIGHTENED_VMCS_RECOMMENDED &&




	    (ms_hyperv.nested_features & HV_X64_ENLIGHTENED_VMCS_VERSION) >=




	    KVM_EVMCS_VERSION) {




		int cpu;










		/* Check that we have assist pages on all online CPUs */




		for_each_online_cpu(cpu) {




			if (!hv_get_vp_assist_page(cpu)) {




				enlightened_vmcs = false;




				break;




			}




		}










		if (enlightened_vmcs) {




			pr_info("KVM: vmx: using Hyper-V Enlightened VMCS\n");




			static_branch_enable(&enable_evmcs);




		}










		if (ms_hyperv.nested_features & HV_X64_NESTED_DIRECT_FLUSH)




			vmx_x86_ops.enable_direct_tlbflush




				= hv_enable_direct_tlbflush;










	} else {




		enlightened_vmcs = false;




	}




#endif










	r = kvm_init(&vmx_x86_ops, sizeof(struct vcpu_vmx),




		     __alignof__(struct vcpu_vmx), THIS_MODULE);




	if (r)




		return r;










	/*




	 * Must be called after kvm_init() so enable_ept is properly set




	 * up. Hand the parameter mitigation value in which was stored in




	 * the pre module init parser. If no parameter was given, it will




	 * contain 'auto' which will be turned into the default 'cond'




	 * mitigation mode.




	 */




	r = vmx_setup_l1d_flush(vmentry_l1d_flush_param);




	if (r) {




		vmx_exit();




		return r;




	}










#ifdef CONFIG_KEXEC_CORE




	rcu_assign_pointer(crash_vmclear_loaded_vmcss,




			   crash_vmclear_local_loaded_vmcss);




#endif




	vmx_check_vmcs12_offsets();










	return 0;




}




module_init(vmx_init);

	Step 4: Build the updated code :
	-make modules 
-rmmod kvm 
-rmmod kvm_intel 
Step 5: Install virt-manager. Download Ubuntu ISO file and create test VM. 
- sudo apt-get install qemu-kvm libvirt-bin bridge-utils virt-manager 
- sudo apt-get install virt-manager 
Step 6: Open Virtual-Manager and start Virtual Machine. Now try dmesg command in the host system’s terminal.
 - dmesg 
 _____________________________________________________________________________________________________________________________
 Assignment 4: Instrumenting KVM : Difference in Performance Using Nested Paging Vs. Shadow Paging
 _____________________________________________________________________________________________________________________________
Steps To Complete This Assignment: 
Step 1: Run Assignment 3 Code.
Step 2: After VM Boot, Dmesg the host machine and record total count for each exit type.
Step 3: Remove KVM-Intel; rmmod kvm-intel now
Step 4: Reload the kvm-intel module with parameter ept=0 disabling nested paging and enabling shadow paging:
        insmod  /lib/modules/XXX/kernel/arch/x86/kvm/kvm-intel.ko ept=0
Step 5: Boot the same test VM again, and capture the same output from your ‘dmesg’ output.

Note: Write Debug Statements At Every Point of the file CPUID.C To Capture Logs to Differentiate Between Exit Counts Of Nested Paging Vs. Shadow Paging.
      printk("sd> Exit_Reason : ALL , ExitCount : %u\n",eax);
      printk("sd> Exit_Reason : ALL , Time_Taken : %lld\n",temp);
      printk("sd> Exit_Reason : %d , ExitCount : %u\n",(int)ecx,eax);
      printk("sd> Exit_Reason : %d , Time_Taken : %lld\n",(int)ecx,temp);
 Step 6: Take Sample logs and Put it in Submission Doc.
